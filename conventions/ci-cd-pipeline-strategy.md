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
| **PR → main (CI)** | `verify` only — **no tests** | cheap (cached) | backstop on every PR |
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

## Cost levers (apply to whatever still runs in CI)
1. **Turbo + bun cache** in CI (`actions/cache` for `.turbo` and the bun store, keyed on
   `bun.lock`) — same checks, 3–5× faster. Biggest single win.
2. **`concurrency: cancel-in-progress`** per PR — superseded pushes don't keep running.
3. **`paths` filters** — a docs-only PR shouldn't run native/backend jobs.
4. **Merge queue** (optional, structural) — validate once at the front of the queue against
   the real pre-merge state, instead of re-validating a moving `main` on every push. Pairs
   with required checks = real branch protection on the one `main`.
5. **Self-hosted runner** (optional, nuclear) — an always-on machine (e.g. a Mac mini)
   runs CI for $0 GitHub minutes. Private repo → acceptable risk.

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
