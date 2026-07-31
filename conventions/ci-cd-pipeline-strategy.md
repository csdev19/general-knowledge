# CI/CD: trunk + tags, one `verify` gate, tiered checks

_How to keep a single `main` trustworthy without burning your GitHub Actions minutes:
run the cheap gate everywhere (local + PR), run the expensive gate only when you
promote. Distilled from a real cost blow-up (a burst of stacked PRs + many merges
exhausted the monthly Actions minutes)._

## The mental model

- **`main` is not prod.** `main` is the current state of the code (integration).
- **Prod is a `tag`.** You promote by cutting a tag, not by merging.
- **Trunk-based: `main` + tags, no `dev` branch.** A `dev` branch would mean three
  flows (dev + main + tags) for a small team — more merges, more CI, more drift.
  Fewer branches = less pain. Trunk-based needs discipline; the **local gate + tags
  are that discipline**, without the weight of an extra long-lived branch.

The cost problem is not "too many checks" — several gates protecting one main is
*healthy*. The waste is treating every check as equal and running all of them, cold,
on every push. Fix: **split checks by cost × purpose, and pay for each where it's worth it.**

## One shared gate: `verify`

A single script is the source of truth for "is this code OK to share":

```
bun run verify   # = turbo check-types + lint + format-check + build packages
```

- **NO tests in `verify`.** Full test suites (web + backend + mobile) are slow; running
  them on every push/PR is the main cost sink. Tests run **locally on demand** and in the
  **promotion (tag) pipeline** — not per-PR.
- Cacheable and fast (turbo skips unchanged packages).
- **The same `verify` is used by all three consumers** below — humans, agents, and CI —
  so "green" means the same thing everywhere.

## Tiered gates (where each check runs)

| Stage | What runs | Cost | Who/when |
| --- | --- | --- | --- |
| **Local pre-push hook** | `verify` (types + lint + format + build) | free (your machine) | every human push |
| **Local on-demand** | `bun run test` (web/backend/mobile) | free | before pushing risky changes |
| **PR → main (CI)** | `pr-verify` (`verify` minus native) always + `pr-native` (native type-check) paths-gated — **no tests** | cheap (non-native never installs Expo) | backstop on every PR |
| **Tag → prod (CI)** | full: build + **tests** + EAS native + e2e + security scan + deploy + migrate | expensive, but rare | on promotion |

Rationale: the cheap gate (`verify`) runs *everywhere* so `main` stays releasable and
breakage is caught at the smallest unit (1 PR, 1 cause). The expensive gate runs *only
when you promote*, where the minutes are worth it. Tests are the slow part → local +
tag, never per-PR.

### Why keep a PR backstop (not 100% local)
A local hook is advice, not a guarantee: it's skippable (`--no-verify`), it doesn't run
for autonomous agents unless wired in, and "works on my machine" ≠ "works in CI". So the
PR runs `verify` once as a cheap backstop. It's the same script, so it almost never fails
if the local hook ran.

## The local pre-push hook

- Commit a hook the whole team inherits: `.githooks/pre-push` → runs `bun run verify`.
  Enable with `git config core.hooksPath .githooks` (documented in the repo README/setup).
- **Pre-push, not pre-commit** — pre-commit fires on every tiny commit and gets bypassed;
  pre-push fires once, when you actually share.
- Keep it fast (turbo cache) or it gets skipped.

## Agents run the same gate
Autonomous workers (the night-runner / headless agents) MUST run `bun run verify` before
pushing — the exact same script as humans and CI. One gate, one meaning of "green". See
[[ai-agent-delegation]].

## The mobile/native install is the biggest single cost — split it out

In an Expo/React Native monorepo the dominant CI cost is **not the build and not the
tests — it's `bun install` of the native app's dependencies.** Measured on trip-planner
with the `deps:weight apps` script below (Bun 1.3, macOS/mini, `--filter` on install):

| Install | Size | Files |
| --- | --- | --- |
| Full (`bun install`) | 1.97 GB | 123k |
| Non-native (`bun install --filter '!native'`) | **1.71 GB** | 107k |
| Native only (`bun install --filter native`) | 1.18 GB | 74k |
| Web only (`--filter web`) | 1.37 GB | 91k |
| Docs only (`--filter docs`) | 0.42 GB | 22k |

Two honesty notes, both learned by *actually measuring* rather than trusting a figure:

1. **Numbers are platform-dependent — measure on the target.** React Native ships Android
   build artifacts inside its npm tarballs (e.g. a `libreactnative.so`/`libhermesvm.so`
   under `android/build/…`). On **macOS** those largely do not materialise, so the mini
   shows native adding only ~0.26 GB over `!native`. On a **Linux CI runner** the native
   tree is materially larger (it writes those Android `.so` files and uses none of them).
   So the split's payoff is *bigger on CI than the mini table suggests* — but treat any
   single number as environment-specific and re-measure where it runs. (An earlier
   "4.6 GB / 195k" figure quoted for the full install was never reproduced on the mini —
   exactly the kind of stale claim the weight script exists to catch.)
2. **The cost is *materialising* files, not fetching them.** A `~/.bun/install/cache` step
   restored 533 MB in 16s to save only ~23s of a ~230s install → **not worth it**. Caching
   the download does little when the time goes into writing 100k+ files.

**The split (two PR jobs):**

- **`pr-verify`** (every PR): `bun install --frozen-lockfile --filter '!native'` then
  `oxlint && oxfmt --check && turbo run check-types build --filter='!native'`.
  **Never installs Expo.** `oxlint`/`oxfmt` read *source files*, not `node_modules`, so
  native source is still linted/formatted here without its deps. (Note: `apps/native`
  has **no `build` task** anyway, so `turbo build` never touches the mobile app.)
- **`pr-native`** (paths-gated): `on.pull_request.paths` lists `apps/native/**` **and
  every workspace `turbo check-types --filter=native` pulls in** (backend, config,
  domain, `bun.lock`). Installs with `--filter native`, runs native's own `tsc`
  (`working-directory: apps/native`, since a `--filter native` install has no root
  turbo). **Type-check only — no tests.** This is the ONLY job that catches a
  shared-package change breaking native, so its `paths` list must stay complete.

**Net:** a non-native PR (the common case, docs included) never pays the Expo install.
**Tradeoff (accept it consciously):** a non-native PR that breaks native *through a
shared package not in `pr-native`'s paths* is caught at tag, not per-PR. Widening
coverage = one line in `paths`.

Proof it works (measured on the mini, warm store): `--filter '!native'` install →
1.71 GB / 107,176 files, **zero** `apps/native/node_modules`, **zero** `react-native`/
`expo`, **zero** `.so` files; full gate green (oxlint 0/0 on 492 files, oxfmt clean on
863, `turbo check-types build --filter='!native'` 12/12).

## Protocol: watch dependency weight (don't let a fat dep slip in)

Big dependencies are the thing that makes installs — and therefore CI — expensive, so
make their weight *visible* instead of discovering it after the minutes are gone. Ship a
small audit script and run it before adding or bumping a heavy dep.

`scripts/dep-weight.sh` (wired as `bun run deps:weight`), two modes:

- **`deps:weight`** (TOP, instant): reads the current `node_modules` and prints the
  heaviest packages + total, flagging anything over `THRESHOLD_MB` (default 50). Answers
  "what is fat right now?" — on trip-planner it surfaced `typescript` (243 MB), `next` +
  `mermaid` + `lucide-react` (docs), `expo-image` (133 MB), `convex` (111 MB),
  `date-fns` (80 MB).
- **`deps:weight apps`** (installs in a throwaway git worktree, slow): reports the install
  footprint (size + file count) of `full`, each `apps/*` via `--filter <app>`, and
  `!native`. Answers "what does each app cost to install, and what does the PR gate
  actually skip?" — this is what produced the table above.

Discipline: before adding/upgrading a dependency, run `deps:weight`; if it lands a package
over the threshold, justify it or find a lighter one (e.g. a 80 MB date lib is a smell —
prefer date-fns subpath imports, dayjs, or `Intl`). Keep heavy, platform-specific trees
(native/Expo) isolated to their own workspace so the `--filter` split keeps working.
The script is generic to any Bun workspace monorepo — copy it into a new repo as-is.

## Cost levers (apply to whatever still runs in CI)
1. **Turbo cache** in CI (`actions/cache` for `.turbo`, keyed on `${{ github.sha }}`
   with a `turbo-` restore prefix) — replays unchanged `check-types`/`build` outputs.
   A **bun install cache** (`~/.bun/install/cache` keyed on `bun.lock`) sounds like the
   same win but was **measured not worth it** for the native tree (see above): the cost
   is materialising files, not downloading them. *Cache is a deliberate, separate step —
   ship a clean no-cache baseline first to measure the real per-PR cost, then add it.*
2. **`concurrency: cancel-in-progress`** per PR — superseded pushes don't keep running.
3. **`paths` filters** — a docs-only PR shouldn't run native/backend jobs. (This is what
   makes `pr-native` cheap — it only fires when native or its deps change.)
4. **Merge queue** (optional, structural) — validate once at the front of the queue against
   the real pre-merge state, instead of re-validating a moving `main` on every push. Pairs
   with required checks = real branch protection on the one `main`.
5. **Self-hosted runner** (optional, nuclear) — an always-on machine (e.g. a Mac mini)
   runs CI for $0 GitHub minutes. **The silver bullet for the native install specifically:**
   on a persistent runner `node_modules` + the bun store *survive between runs*, so the
   Expo install is near-instant after the first time (no re-materialising). Private repo →
   acceptable risk. Needs a runner registration token from the repo owner.

## Deploy ordering (a trap worth documenting)
`convex deploy` (schema) runs before `migrations:runAll` in the release pipeline. A
schema change that narrows a validator will FAIL the deploy if existing rows violate it —
before the migration can fix them. **Two-step promotion:** release once that runs the
data migration while the schema still accepts old values, then release the narrowing.
Use the pipeline's existing `db:migrate`; don't invent a second path.

## Minimal branch protection
Require only `verify` on `main` (via the PR backstop / merge queue). That alone stops
red-merges (the failure mode of "no protection": anything can land on the one main).

## Checklist to adopt this in a repo
- [ ] Add `verify` script (types + lint + format + build; NO tests) to root `package.json`.
- [ ] Add `.githooks/pre-push` running `bun run verify`; document `core.hooksPath`.
- [ ] PR workflow: run `verify` only, cached; drop tests from PR.
- [ ] Tag workflow(s): build + tests + native builds + e2e + security + deploy + migrate.
- [ ] Turbo/bun cache + `cancel-in-progress` + `paths` on the surviving jobs.
- [ ] Agents call `bun run verify` before push.
- [ ] Require `verify` on `main`.

## See also
- [[ai-agent-delegation]] — agents share the same `verify` gate.
- [[specs-and-plans-workflow]] · [[plan-to-backlog]] — how this gets specced and shipped.
- [[changelog-pattern]] — releases (tags) are where the changelog + heavy pipeline land.
