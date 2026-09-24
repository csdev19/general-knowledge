# Release-gated verification

_Where each check runs — local push, release candidate, or release — and the one rule that
is not negotiable: **a workflow that publishes never runs its publish step without a
verification job on the same SHA in front of it.** Written after the same account ran out
of GitHub Actions quota twice in five days (2026-09-19 and 2026-09-23) and a release PR
stopped updating, with a strategy doc that already said most of the right things._

This is the normative version. [ci-cd-pipeline-strategy.md](../conventions/ci-cd-pipeline-strategy.md)
has the mental model and the native-install split; [ci-runner-cost.md](./ci-runner-cost.md)
has the cost model per runner OS. This page says **what must be true in any repository**,
how to audit an existing one in five minutes, and what to copy.

## Why the previous guidance did not prevent it

The account had a written strategy: trunk + tags, one `verify` gate, tests at tag time.
Two repositories still burned 802 and 244 Linux minutes plus 109 macOS minutes on PR-time
checks in 23 days, while their release workflows published with **no** verification job at
all. Four gaps, each now a rule below:

1. The release gate was a checklist item, not a precondition. The PR checks were added
   first because they were visible; the tag workflow "would get tests later". Later never came.
2. The guidance assumed a cheap PR backstop was free. On a single-owner account with no
   branch protection the backstop is advisory, and it still bills every push.
3. Nobody measured the local gate before deciding tests were "the slow part". Measured,
   the whole suite took 18 s locally and 208 s on a runner.
4. Bot workflows were invisible. `release-please` ran on every push to `main`, docs merges
   included, at a billed minimum of one minute each: 90 minutes, and the first job blocked
   when the quota ran out.

## The rules

**R1 — Publishing is gated, always.** Every workflow that deploys, uploads an installer,
writes an update feed, or creates a release has a `verify` job that the publishing job
`needs:`. It runs on the exact tagged SHA in a clean environment. This holds whether or not
PR checks exist. Audit: every file under `.github/workflows/` that mentions `deploy`,
`s3 cp`, `wrangler`, `gh-release`, `eas`, `publish` or `upload` must contain `needs:`.

**R2 — One definition of verified.** A root `verify` script is the only thing hooks and
workflows call. Lint, read-only format check, type-check, package builds, tests. If a check
is not in `verify`, it is not part of "green".

**R3 — Budget before workflows.** Set a non-zero Actions spending limit and the 75/90/100 %
alerts before touching a single workflow. With no payment method or a zero limit, exhausting
the quota **blocks jobs** rather than billing; the first job blocked is whichever runs next,
usually the release. The quota is per account, so one noisy repository blocks another's release.

**R4 — Measure the local gate first.** Run `verify` cold and warm on the machine that pushes.
If it is under a minute warm, every check in it belongs in the pre-push hook, and PR-time
cloud runs are optional. Do not assume tests are slow; the runner is what is slow.

**R5 — Every workflow header states its tier and its reopen condition.** A future reader
must see that a trigger was chosen, not inherited.

**R6 — Bots are paid too.** Every job bills a whole minute minimum. Path-filter bot
workflows (`release-please`, dependabot helpers, labelers) to the paths that can change
their outcome, and give them `workflow_dispatch` for manual catch-up.

**R7 — Never move a platform-gated suite to a cheaper runner.** A suite whose tests skip
off-platform passes without testing; see the trap in
[ci-runner-cost.md](./ci-runner-cost.md#the-trap-a-cheaper-runner-can-make-a-suite-pass-without-testing-anything).
Platform E2E runs on the platform, at release time, against the bundle that ships.

**R8 — Keep the evidence.** Archive the billing export before changing anything, and
the next one after. Otherwise "we saved 80 %" is a feeling.

## Where each check lives

Three tiers. Each check has one home, marked ●, and at most one echo, marked ○.

| Check                                | Local push (hook) | Candidate (release PR / dispatch) |   Release (tag)   |
| ------------------------------------ | :---------------: | :-------------------------------: | :---------------: |
| Lint, format check                   |         ●         |                 ○                 |         ●         |
| Type-check, all workspaces           |         ●         |                 ○                 |         ●         |
| Package builds                       |         ●         |                 ○                 |         ●         |
| Unit and component tests             |         ●         |                 ○                 |         ●         |
| App builds (web, API bundle)         |                   |                 ○                 |         ●         |
| Platform E2E (Electron, mobile)      |     on demand     |                                   |         ●         |
| Native rebuilds, signing, notarizing |                   |                                   |         ●         |
| Security scan, dependency audit      |                   |                 ○                 |         ●         |
| Deploy, upload, migrate              |                   |                                   | ● after all above |

- **Local push** is the Lefthook `pre-push` running `verify`. Free, seconds, bypassable.
- **Candidate** is the release PR that release-please maintains, or a manual
  `workflow_dispatch` of the validation workflow. It runs `verify` on `main` **before the
  tag exists**, so a failure costs a fix and a push, not a tag and a patch release. Cheap:
  one Linux job per release-PR update, not per feature-PR push.
- **Release** is the tag workflow. `verify` job first, then platform E2E against the built
  bundle, then publish. Enforced by `needs:`, not by habit.

Per-feature-PR cloud checks are a fourth, optional tier. They earn their minutes only when
(a) branch protection makes them required, because then they are a guarantee instead of a
notification, or (b) more than one person merges. A single owner with no protection is
paying for a status icon.

## Pick the tier by who merges

| Situation                                   | PR-time cloud checks               | Candidate | Release gate |
| ------------------------------------------- | ---------------------------------- | :-------: | :----------: |
| One owner, no branch protection             | Off; hook + candidate replace them |     ●     |      ●       |
| One owner, agents pushing on their behalf   | Off; agents run `verify` too       |     ●     |      ●       |
| Two or more people merging                  | `verify` only, required, cached    |     ●     |      ●       |
| Public repository with outside contributors | `verify` only, required            |     ●     |      ●       |

The release gate is in every row. That column is what R1 means.

## What you give up without PR-time cloud checks, and how to close it

Be honest about it when writing the ADR:

1. **Clean-environment detection.** The hook runs with the developer's cache, binaries and
   working tree. It cannot see an uncommitted file, a stale lockfile, a native dependency that
   only fails on a fresh install, or an environment variable that exists locally and not on a
   runner. Close it with a **risky-change list**: any PR touching the lockfile, `scripts/`,
   workflows, native dependencies or package build config gets one manual dispatch of the
   validation workflow before merge. Minutes per use, not per push.
2. **Bisection.** A red candidate after ten merges has ten suspects. The candidate tier keeps
   the window short; a dispatch after a risky merge keeps it shorter.
3. **Bypass.** `LEFTHOOK=0` exists and the hook attests to a working tree, not to the pushed
   SHA. Optional guard: refuse to push with uncommitted changes to tracked files.
4. **Timing.** Problems surface when preparing a release, not when opening a PR. The
   candidate tier is what makes that acceptable: the tag does not exist yet.

## Copy these

Root `package.json`:

```json
"verify": "bun run lint && oxfmt --check . && bun run build --filter='@scope/*' && bun run check-types && bun run test"
```

`lefthook.yml` (pre-commit and commit-msg unchanged, see the Lefthook playbook):

```yaml
pre-push:
  jobs:
    - name: verify
      run: bun run verify
```

Release workflow gate, in every `release-*.yml`:

```yaml
jobs:
  verify:
    name: Verify (lint, types, tests)
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v2
        with: { bun-version: 1.3.4 }
      - uses: actions/cache@v4
        with:
          path: ~/.bun/install/cache
          key: ${{ runner.os }}-bun-${{ hashFiles('bun.lock') }}
      - run: bun install --frozen-lockfile
      - run: bun run verify

  release-<component>:
    needs: verify
    # build → platform E2E against that build → sign → publish, in that order
```

Candidate tier on the release PR (the bot's branch prefix is `release-please--`):

```yaml
on:
  pull_request:
    branches: [main]
jobs:
  verify:
    if: startsWith(github.head_ref, 'release-please--')
    # same steps as above; skipped jobs on other PRs cost nothing
```

Bot workflow filter:

```yaml
on:
  push:
    branches: [main]
    paths:
      - "apps/**"
      - "packages/**"
      - "bun.lock"
      - "release-please-config.json"
      - ".release-please-manifest.json"
  workflow_dispatch:
```

Retired validation workflows keep `workflow_dispatch:` only and say why in the header.

## Five-minute audit of an existing repository

- [ ] `grep -L "needs:" .github/workflows/release-*.yml` prints nothing. Anything printed
      publishes unverified: **fix this before anything else**.
- [ ] A root `verify` script exists and the pre-push hook calls it.
- [ ] `time bun run verify` twice; write both numbers in the repo's ADR.
- [ ] Every workflow with `pull_request:` either is required by branch protection or has a
      header sentence justifying its minutes.
- [ ] `release-please.yml` (or any bot) has a `paths` filter.
- [ ] Every non-Linux runner is in a release workflow, or its header says which ladder rung
      it is on and why.
- [ ] Billing → Budgets: Actions limit is non-zero, alerts on.
- [ ] The last billing export is archived in the repo with a checksum.

## Measured example

Kaipu, September 2026, 23 days, single owner, three deployables (desktop, web, API),
13 releases in the period:

| Item                                  | Before        | After (projected) |
| ------------------------------------- | ------------- | ----------------- |
| PR-time Linux minutes, four workflows | 802           | 0                 |
| `release-please` Linux minutes        | 90            | 45–55             |
| Release `verify` jobs, Linux          | 0             | 65–90             |
| Release macOS minutes                 | 42.5          | 50–55 (E2E added) |
| Repository gross                      | $8.09         | $4.1–4.4          |
| Release publishes without a gate      | every release | none              |

Local `verify`: 27 s cold, 3 s warm. Desktop suite: 17 s locally, 208 s on `ubuntu-latest`.
The projection is written down before the "after" export exists; replace it when it does.

## Related

- [ci-cd-pipeline-strategy.md](../conventions/ci-cd-pipeline-strategy.md) — the mental model
  (main is not prod, prod is a tag) and the native-install split.
- [ci-runner-cost.md](./ci-runner-cost.md) — runner-OS multipliers, the escalation ladder,
  the platform-gate trap, measuring with the REST API.
- [pr-checks.md](./pr-checks.md) — the shape of PR-time checks when you do keep them, and
  why a `paths`-filtered check must not be required.
- [release-please-playbook.md](./release-please-playbook.md) — the release PR this page
  turns into the candidate tier.
- Lefthook playbook — pre-commit, commit-msg and the migration from husky. Merged in hub
  PR #19 onto the `docs/english-only` branch, not yet on `main`.
