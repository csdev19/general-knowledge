# CI/CD: trunk + tags, one `verify` gate, tiered checks

_How to keep a single `main` trustworthy without burning your GitHub Actions minutes:
run the cheap gate everywhere (local + PR), run the expensive gate only when you
promote. Distilled from a real cost blow-up (a burst of stacked PRs + many merges
exhausted the monthly Actions minutes)._

> **Read [release-gated-verification.md](../monorepos/release-gated-verification.md) first
> (2026-09-24).** It is the normative version of this page after the same account exhausted
> its Actions quota twice with this strategy written down. Two things below are corrected
> there: the release gate is a **precondition** of every publish step (not a checklist item),
> and "no tests in `verify`" only holds if you measured them as slow — on a Bun + Turbo
> monorepo the full suite ran in 18 s locally. The PR backstop is optional by who merges;
> the release gate is not.

## The mental model

- **`main` is not prod.** `main` is the current state of the code (integration).
- **Prod is a `tag`.** You promote by cutting a tag, not by merging.
- **Trunk-based: `main` + tags, no `dev` branch.** A `dev` branch would mean three
  flows (dev + main + tags) for a small team — more merges, more CI, more drift.
  Fewer branches = less pain. Trunk-based needs discipline; the **local gate + tags
  are that discipline**, without the weight of an extra long-lived branch.

The cost problem is not "too many checks" — several gates protecting one main is
_healthy_. The waste is treating every check as equal and running all of them, cold,
on every push. Fix: **split checks by cost × purpose, and pay for each where it's worth it.**

## One shared gate: `verify`

A single script is the source of truth for "is this code OK to share":

```
bun run verify   # = turbo check-types + lint + format-check + build packages
```

- **Tests in `verify` only if they are fast — measure first.** Measured on a Bun + Turbo
  monorepo, 1,167 tests ran in 18 s cold and were cached warm, so they belong in the hook.
  Where a suite is genuinely slow (mobile, integration against a database), it runs
  **locally on demand** and in the **promotion (tag) pipeline** — never per feature-PR push.
- Cacheable and fast (turbo skips unchanged packages).
- **The same `verify` is used by all three consumers** below — humans, agents, and CI —
  so "green" means the same thing everywhere.

## Tiered gates (where each check runs)

| Stage                   | What runs                                                                                               | Cost                                   | Who/when                     |
| ----------------------- | ------------------------------------------------------------------------------------------------------- | -------------------------------------- | ---------------------------- |
| **Local pre-push hook** | `verify` (types + lint + format + build + fast tests)                                                   | free (your machine)                    | every human push             |
| **Local on-demand**     | the suites too slow for the hook (mobile, integration)                                                  | free                                   | before pushing risky changes |
| **PR → main (CI)**      | `pr-verify` (`verify` minus native) always + `pr-native` (native type-check) paths-gated — **no tests** | cheap (non-native never installs Expo) | backstop on every PR         |
| **Tag → prod (CI)**     | full: build + **tests** + EAS native + e2e + security scan + deploy + migrate                           | expensive, but rare                    | on promotion                 |

Rationale: the cheap gate (`verify`) runs _everywhere_ so `main` stays releasable and
breakage is caught at the smallest unit (1 PR, 1 cause). The expensive gate runs _only
when you promote_, where the minutes are worth it. Whether tests sit in the cheap gate is
a measurement, not a rule — see the `verify` bullet above; only the suites you measured as
slow drop to local-on-demand + tag.

### Why keep a PR backstop (not 100% local)

A local hook is advice, not a guarantee: it's skippable (`--no-verify`), it doesn't run
for autonomous agents unless wired in, and "works on my machine" ≠ "works in CI". So the
PR runs `verify` once as a cheap backstop. It's the same script, so it almost never fails
if the local hook ran.

## The local pre-push hook

- Commit a hook the whole team inherits. **[lefthook](https://lefthook.dev)** is the tool
  of choice: one `lefthook.yml` in the repo, installed by `bun install` (it is a devDep
  with no compile step), no `core.hooksPath` to remember. The hand-rolled
  `.githooks/` + `git config core.hooksPath` route works too and needs no dependency.
- **The contract, per hook:** `pre-commit` fixes what you commit (format `--write`, lint
  on staged files — cheap, never blocks); `commit-msg` rejects what release automation
  cannot parse; **`pre-push` runs `bun run verify`** — the _same_ script CI runs.
- **Pre-push is where the cost is saved, not pre-commit** — pre-commit fires on every
  tiny commit and gets bypassed; pre-push fires once, when you actually share. A push
  that fails types in CI is a whole CI run wasted, then a second one after the fix.
- **Run the affected tests there too**, free: `turbo run test --filter='...[origin/main]'`
  only runs the packages the push touched, from cache. This is what earns the right to
  take tests off the PR path (tiered gates above) — without a local test gate, dropping
  PR tests just moves breakage to `main`.
- Keep it fast (turbo cache) or it gets skipped. `LEFTHOOK=0 git push` is the documented
  bypass; the PR `verify` backstop is why a bypass is tolerable.

A repo that runs lint + format in pre-push but leaves types and build to CI has the tiers
inverted: the free gate is weaker than the paid one. Audit the hook, not just the
workflows.

## Agents run the same gate

Autonomous workers (the night-runner / headless agents) MUST run `bun run verify` before
pushing — the exact same script as humans and CI. One gate, one meaning of "green". See
[[ai-agent-delegation]].

## The mobile/native install is the biggest single cost — split it out

In an Expo/React Native monorepo the dominant CI cost is **not the build and not the
tests — it's `bun install` of the native app's dependencies.** Measured on trip-planner
with the `deps:weight apps` script below (Bun 1.3, macOS/mini, `--filter` on install):

| Install                                       | Size        | Files |
| --------------------------------------------- | ----------- | ----- |
| Full (`bun install`)                          | 1.97 GB     | 123k  |
| Non-native (`bun install --filter '!native'`) | **1.71 GB** | 107k  |
| Native only (`bun install --filter native`)   | 1.18 GB     | 74k   |
| Web only (`--filter web`)                     | 1.37 GB     | 91k   |
| Docs only (`--filter docs`)                   | 0.42 GB     | 22k   |

Two honesty notes, both learned by _actually measuring_ rather than trusting a figure:

1. **Numbers are platform-dependent — measure on the target.** React Native ships Android
   build artifacts inside its npm tarballs (e.g. a `libreactnative.so`/`libhermesvm.so`
   under `android/build/…`). On **macOS** those largely do not materialise, so the mini
   shows native adding only ~0.26 GB over `!native`. On a **Linux CI runner** the native
   tree is materially larger (it writes those Android `.so` files and uses none of them).
   So the split's payoff is _bigger on CI than the mini table suggests_ — but treat any
   single number as environment-specific and re-measure where it runs. (An earlier
   "4.6 GB / 195k" figure quoted for the full install was never reproduced on the mini —
   exactly the kind of stale claim the weight script exists to catch.)
2. **The cost is _materialising_ files, not fetching them.** A `~/.bun/install/cache` step
   restored 533 MB in 16s to save only ~23s of a ~230s install → **not worth it**. Caching
   the download does little when the time goes into writing 100k+ files.

**The split (two PR jobs):**

- **`pr-verify`** (every PR): `bun install --frozen-lockfile --filter '!native'` then
  `oxlint && oxfmt --check && turbo run check-types build --filter='!native'`.
  **Never installs Expo.** `oxlint`/`oxfmt` read _source files_, not `node_modules`, so
  native source is still linted/formatted here without its deps. (Note: `apps/native`
  has **no `build` task** anyway, so `turbo build` never touches the mobile app.)
- **`pr-native`** (paths-gated): `on.pull_request.paths` lists `apps/native/**` **and
  every workspace `turbo check-types --filter=native` pulls in** (backend, config,
  domain, `bun.lock`). Installs with `--filter native`, runs native's own `tsc`
  (`working-directory: apps/native`, since a `--filter native` install has no root
  turbo). **Type-check only — no tests.** This is the ONLY job that catches a
  shared-package change breaking native, so its `paths` list must stay complete.

**Net:** a non-native PR (the common case, docs included) never pays the Expo install.
**Tradeoff (accept it consciously):** a non-native PR that breaks native _through a
shared package not in `pr-native`'s paths_ is caught at tag, not per-PR. Widening
coverage = one line in `paths`.

Proof it works (measured on the mini, warm store): `--filter '!native'` install →
1.71 GB / 107,176 files, **zero** `apps/native/node_modules`, **zero** `react-native`/
`expo`, **zero** `.so` files; full gate green (oxlint 0/0 on 492 files, oxfmt clean on
863, `turbo check-types build --filter='!native'` 12/12).

## Protocol: watch dependency weight (don't let a fat dep slip in)

Big dependencies are the thing that makes installs — and therefore CI — expensive, so
make their weight _visible_ instead of discovering it after the minutes are gone. Ship a
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

## Runner cost and path fan-out (read this first)

Where a job runs matters more than how long it takes: `macos-latest` bills at ~10× Linux, so
the workflow that runs least can be the largest line on the bill. The cost model, the
multiplier table, the spending-limit ceiling, the `packages/**` shared-filter trap and the
per-PR → on-merge → nightly → release ladder are all in
**[monorepos/ci-runner-cost.md](../monorepos/ci-runner-cost.md)** — one source of truth; not
restated here.

Two smells that doc does not name, both cheap to fix:

- **The same suite owned by two workflows.** A repo-wide `test --filter='*'` plus a per-app
  `test` doubles the most-run job in the repo. One owner per suite.
- **The free gate weaker than the paid one.** If pre-push runs lint + format but leaves types,
  build and affected tests to CI, the tiers above are inverted — see the pre-push section.

## Cost levers (apply to whatever still runs in CI)

1. **Turbo cache** in CI (`actions/cache` for `.turbo`, keyed on `${{ github.sha }}`
   with a `turbo-` restore prefix) — replays unchanged `check-types`/`build` outputs.
   A **bun install cache** (`~/.bun/install/cache` keyed on `bun.lock`) sounds like the
   same win but was **measured not worth it** for the native tree (see above): the cost
   is materialising files, not downloading them. _Cache is a deliberate, separate step —
   ship a clean no-cache baseline first to measure the real per-PR cost, then add it._
2. **`concurrency: cancel-in-progress`** per PR — superseded pushes don't keep running.
3. **`paths` filters** — a docs-only PR shouldn't run native/backend jobs. (This is what
   makes `pr-native` cheap — it only fires when native or its deps change.)
4. **Merge queue** (optional, structural) — validate once at the front of the queue against
   the real pre-merge state, instead of re-validating a moving `main` on every push. Pairs
   with required checks = real branch protection on the one `main`.
5. **Self-hosted runner** (optional, nuclear) — an always-on machine (e.g. a Mac mini)
   runs CI for $0 GitHub minutes. **The silver bullet for the native install specifically:**
   on a persistent runner `node_modules` + the bun store _survive between runs_, so the
   Expo install is near-instant after the first time (no re-materialising). Private repo →
   acceptable risk. Needs a runner registration token from the repo owner.

## Deploy ordering (a trap worth documenting)

`convex deploy` (schema) runs before `migrations:runAll` in the release pipeline. A
schema change that narrows a validator will FAIL the deploy if existing rows violate it —
before the migration can fix them. **Two-step promotion:** release once that runs the
data migration while the schema still accepts old values, then release the narrowing.
Use the pipeline's existing `db:migrate`; don't invent a second path.

## Minimal branch protection

Require one check on `main`. That alone stops red-merges — the failure mode of "no
protection" being that anything can land on the one trunk.

**Which check, though, depends on what still runs per PR.** If the PR backstop survived,
require it. If it did not — the normative page's "who merges" question can retire it — then
the only check left that runs on every PR is the release-candidate **gate job**, and that is
the one to require. Requiring a check that does not always report blocks merges forever
rather than protecting anything.

The mechanism, the `gh api` call, and the two ways enabling it breaks working PRs are in
[monorepos/branch-rulesets.md](../monorepos/branch-rulesets.md). Read it before enabling,
not after: it is retroactive, and it blocks every open PR whose head predates the check.

## Checklist to adopt this in a repo

- [ ] Add a `verify` script (lint + format check + build + types + **tests**) to root
      `package.json`. The banner at the top of this page corrects the original "NO tests"
      advice, and the checklist now matches it: exclude the tests only if you **measured**
      them as slow. On a Bun + Turbo monorepo the full suite is seconds, and a `verify`
      without tests forces every release gate to bolt its own test step on — which is a
      second definition of "verified", and the thing that drifts.
- [ ] Add a pre-push hook running `bun run verify` (lefthook, or `.githooks/` +
      `core.hooksPath`). One job, not `verify` plus a separate affected-tests job.
- [ ] PR workflow, **if you keep one**: run `bun run verify`, cached. Whether to keep it is
      the "who merges" question in the normative page, not a default.
- [ ] Tag workflow(s): a `verify` job that the publish job `needs:` — **do this before removing
      anything from PRs** — then build + tests + native builds + e2e + security + deploy + migrate.
- [ ] Non-zero Actions spending limit and usage alerts on the account.
- [ ] Bot workflows (`release-please`) path-filtered; every job bills a whole minute.
- [ ] Turbo/bun cache + `cancel-in-progress` + `paths` on the surviving jobs.
- [ ] Audit the runner of every job: nothing on `macos`/`windows` that Linux can run.
- [ ] Check no shared-package glob (`packages/**`) fans one PR out to every app workflow.
- [ ] Check no suite runs in two workflows.
- [ ] Agents call `bun run verify` before push.
- [ ] Require a check on `main` that runs on **every** PR to it — see
      [monorepos/branch-rulesets.md](../monorepos/branch-rulesets.md). Pin it to the Actions
      app, list the open PRs first, and expect each of them to need one push afterwards.
- [ ] Never read a stacked PR's empty check list as a pass: `branches:` in a `pull_request`
      trigger filters the PR's **base**, so a PR targeting anything but the trunk runs no
      workflow at all.

## See also

- [[ai-agent-delegation]] — agents share the same `verify` gate.
- [[specs-and-plans-workflow]] · [[plan-to-backlog]] — how this gets specced and shipped.
- [[changelog-pattern]] — releases (tags) are where the changelog + heavy pipeline land.
