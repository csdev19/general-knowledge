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
Platform E2E runs on the platform, against the bundle that ships: on a ready candidate for
that platform, and again at release time.

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
| Platform E2E (Electron, mobile)      |     on demand     |   ● that platform's candidates    |         ○         |
| Native rebuilds, signing, notarizing |                   |                                   |         ●         |
| Security scan, dependency audit      |                   |                 ○                 |         ●         |
| Deploy, upload, migrate              |                   |                                   | ● after all above |

- **Local push** is the Lefthook `pre-push` running `verify`. Free, seconds, bypassable.
- **Candidate** is the release PR that release-please maintains, or a manual
  `workflow_dispatch` of the validation workflow. The bot opens the PR as a **draft**
  (`draft-pull-request: true`) and rewrites it on every releasable merge; a draft allocates
  no runner. Pressing **Ready for review** is the signal "this is the one I am publishing":
  from then on every revision runs `verify`, plus the platform E2E when the candidate is for
  that platform. It all runs **before the tag exists**, so a failure costs a fix and a push,
  not a tag and a patch release. See
  [pr-checks.md](./pr-checks.md#draft--ready-for-review-as-the-trigger) for the workflow shape.
- **Release** is the tag workflow. `verify` job first, then platform E2E against the built
  bundle, then publish. Enforced by `needs:`, not by habit. The E2E stays here as well as in
  the candidate: the tagged SHA is a merge commit, not the candidate's head, and R1 is about
  the SHA that ships.

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
| Public repository, collaborators-only PRs   | Off; immediate pass (see below)    |     ●     |      ●       |
| Public repository with outside contributors | `verify` + platform E2E, required  |     ●     |      ●       |

The release gate is in every row. That column is what R1 means.

In every row, **draft means "do not spend runners yet"**: PR-time checks, where they exist,
run only on non-draft PRs. The outside-contributors row is different because
GitHub-hosted runners, macOS included, are free for public repositories, and strangers do
not run your hooks. So every non-draft PR runs `verify` and the platform E2E, and the
pre-push hook shrinks to the fast checks (lint, format, types). Do not copy that row while
the repository is private: macOS bills at about ten times Linux, which is the problem this
page was written after.

**Free minutes are not free time.** A public repository whose pull requests come only from
collaborators gets nothing from per-PR runs that the hook and the candidate do not already
give, and pays in waiting: every merge, every layer of a stack. Use the collaborators-only
row, and move to the outside-contributors row the day strangers' pull requests are accepted.

## A required check with candidate-only verification

Once branch protection or a ruleset **requires** a check, "verify only the candidate" needs
one more piece: every other pull request must still get that check reported, or it sits on
"Expected" forever and cannot merge. A job-level `if: startsWith(github.head_ref, …)` is not
enough on its own: a job skipped by its `if` reports `skipped`, and a workflow that never
starts reports nothing at all.

The shape that works:

1. **A classify job runs first on every non-draft revision.** It is cheap (seconds, no
   checkout needed) and outputs `release` — the head branch matches the bot's prefix
   (`release-please--branches--*`) or the run is a manual dispatch — plus `docs_only` and
   `e2e` when the candidate tier skips the platform E2E for documentation.
2. **Every verifying job `needs:` the classify job** and runs only when it classified a
   candidate. An ordinary pull request starts no verifying runner.
3. **The required gate reports from a decision, never from the event shape.** It runs when
   the classification says "not a candidate" (and passes at once), when the classification
   failed (and fails), or when a candidate's verification reached a verdict. A draft, a
   cancelled classification or a superseded run reports **nothing** — GitHub takes the worst
   conclusion across runs on a commit, so a red from a run that decided nothing outlives the
   green that follows it. See [pr-checks.md](./pr-checks.md#making-a-sometimes-skipped-job-a-required-check).
4. **No `branches:` filter on `pull_request`.** In a GitHub stack every pull request targets
   the branch below it, yet each is judged against the stack base's required check. Filter to
   `main` and the upper layers never run, so **Merge stack** never turns ready. With the
   classify-and-pass shape the cost of running on every base is one short job.
5. **Simulate the conditions in a script, in `verify`.** The gate's correctness lives in
   `if:` expressions and a shell step that no unit test reaches. A small checker that parses
   the workflow, evaluates each job's `if:` against scenarios (ordinary, stacked, draft,
   candidate, superseded, failed classification, dispatch) and runs the gate's real script
   against a decision table catches the regressions that otherwise ship as a stuck or a
   falsely green check. Derive each scenario's classification from its event, never from the
   expectation, or the check agrees with itself.

What it costs, in the ADR: `main` can be red between releases; a broken test surfaces on the
release pull request, possibly several merges after its cause; and outside contributions are
unverified. The section above on what you give up applies unchanged. Reopen when a release
pull request fails on a regression that takes more than one bisect to place (first lever:
unit tests in the pre-push hook), when strangers' pull requests are accepted, or when a second
person merges regularly.

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

Candidate tier with a **required** check (the shape described above; condensed, the
eligibility checker and the E2E job are omitted):

```yaml
on:
  pull_request: # no `branches:` filter, so every layer of a stack reports
    types: [opened, reopened, synchronize, ready_for_review, converted_to_draft]
  workflow_dispatch:
jobs:
  changes:
    name: Classify the change
    if: github.event_name == 'workflow_dispatch' || github.event.pull_request.draft == false
    runs-on: ubuntu-latest
    outputs:
      release: ${{ steps.classify.outputs.release }}
    steps:
      - id: classify
        env:
          EVENT: ${{ github.event_name }}
          HEAD_REF: ${{ github.head_ref }}
        run: |
          release=false
          case "$EVENT:$HEAD_REF" in
            workflow_dispatch:* | pull_request:release-please--branches--*) release=true ;;
          esac
          echo "release=$release" >> "$GITHUB_OUTPUT"

  verify:
    needs: [changes]
    if: needs.changes.result == 'success' && needs.changes.outputs.release == 'true'
    # … the steps from the release gate above

  gate: # the REQUIRED check
    name: Merge requirements
    needs: [changes, verify]
    if: >-
      always() && github.event_name == 'pull_request' && (
        (needs.changes.result == 'success' && needs.changes.outputs.release == 'false') ||
        needs.changes.result == 'failure' ||
        needs.verify.result == 'success' || needs.verify.result == 'failure'
      )
    runs-on: ubuntu-latest
    steps:
      - env:
          CHANGES: ${{ needs.changes.result }}
          RELEASE: ${{ needs.changes.outputs.release }}
          VERIFY: ${{ needs.verify.result }}
        run: |
          [ "$CHANGES" = "success" ] || { echo "classification failed"; exit 1; }
          [ "$RELEASE" = "false" ] && { echo "not a release candidate"; exit 0; }
          [ "$VERIFY" = "success" ]
```

Candidate tier without a required check (the bot's branch prefix is `release-please--`;
`"draft-pull-request": true` in `release-please-config.json`):

```yaml
on:
  pull_request:
    branches: [main]
    types: [opened, reopened, synchronize, ready_for_review, converted_to_draft]
jobs:
  verify:
    if: >-
      startsWith(github.head_ref, 'release-please--') &&
      github.event.pull_request.draft == false
    concurrency:
      group: verify-pr-${{ github.event.pull_request.number }}
      cancel-in-progress: true
    # same steps as above; skipped jobs on drafts and other PRs cost nothing
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
- [ ] `release-please-config.json` sets `"draft-pull-request": true`, and no PR workflow
      lists `labeled`, `unlabeled` or `edited` in its `types`.
- [ ] Every platform E2E that runs at the tag also runs on that platform's ready candidate.
- [ ] If a check is required: an ordinary pull request and the upper layer of a stack both
      get it reported (no `branches:` filter; a classify job; the gate passes "not a
      candidate"), and a script in `verify` simulates the gate's conditions.
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

### Back from "every PR" to candidate-only, once the repository was public

Kaipu, 2026-10-19. The repository went public on 2026-10-05 and moved every pull request onto
full verification (its ADR 0010, the outside-contributors row). Two weeks later, with pull
requests still limited to collaborators:

| Measured on a code pull request          | Every PR verified | Candidate-only (ADR 0027) |
| ---------------------------------------- | ----------------- | ------------------------- |
| Wall time to a green required check      | 234 s             | ~10 s                     |
| of which the macOS E2E                   | 210 s             | 0 (release PR only)       |
| Linux checks (parallel, slowest part)    | ~100 s            | 0 (release PR only)       |
| Upper layer of a stack                   | never ran — stuck | ~10 s                     |

Classify 4 s, gate 3 s, alias 3 s on that repository's own PR #350. The stuck stack (its PR
#347) was the `branches: [main]` filter: the upper pull request targeted the lower one's
branch, so the workflow never started and **Merge stack** waited on "Expected" forever. The
pre-push hook kept lint, format, cached builds and types (about 7 s); unit tests stayed off the
push by the owner's choice, named in the ADR as the first lever if a release turns red.

### A tag burnt by an E2E that only ran after it

Kaipu desktop 0.8.0, 2026-09-25. The macOS E2E sat in the tag workflow between `Build` and
`Sign`; it failed on its first click. Result: tag created, GitHub Release published with
zero assets, version number burnt. Installed apps were safe, because the update feed is
written after signing. `verify` had not missed anything: it contains no E2E by design. The
fix was to run the same E2E job on the ready desktop candidate as well, so the next failure
cost a push. Source: that repository's ADR 0006.

## Related

- [ci-cd-pipeline-strategy.md](../conventions/ci-cd-pipeline-strategy.md) — the mental model
  (main is not prod, prod is a tag) and the native-install split.
- [ci-runner-cost.md](./ci-runner-cost.md) — runner-OS multipliers, the escalation ladder,
  the platform-gate trap, measuring with the REST API.
- [pr-checks.md](./pr-checks.md) — the shape of PR-time checks when you do keep them, and
  why a `paths`-filtered check must not be required.
- [release-please-playbook.md](./release-please-playbook.md) — the release PR this page
  turns into the candidate tier.
- [public-repo-production-protection.md](./public-repo-production-protection.md) — which refs
  may reach the secrets the publish step holds, and the settings to audit when a repository
  goes public.
- [dependabot-and-pinned-actions.md](./dependabot-and-pinned-actions.md) — why bump PRs usually
  show skipped checks under this model, and what to do before merging one.
- Lefthook playbook — pre-commit, commit-msg and the migration from husky. Merged in hub
  PR #19 onto the `docs/english-only` branch, not yet on `main`.
