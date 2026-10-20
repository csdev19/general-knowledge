# PR checks — validation & E2E

_The GitHub Actions workflows that run on every PR into main (PR Validation and E2E Desktop), what each does, why they are deliberately separate, and the desktop E2E harness they gate._

Two GitHub Actions workflows run on every pull request into `main`. They are **deliberately
separate** — this page explains what each does, why the split is intentional, and the E2E
harness behind the second one. (For a further evolution of this split into per-project pipelines,
see [ci-per-project-pipelines.md](./ci-per-project-pipelines.md).)

> These are PR-gate checks, not deploys. Releases are tag-driven — see
> [ci-cd-pipelines.md](./ci-cd-pipelines.md).

## The two workflows

| Workflow          | File                                  | Runs     | Scope                            |
| ----------------- | ------------------------------------- | -------- | -------------------------------- |
| **PR Validation** | `.github/workflows/pr-validation.yml` | ~4 min   | Cheap, **monorepo-wide** gate    |
| **E2E Desktop**   | `.github/workflows/e2e-desktop.yml`   | ~1–2 min | Expensive, **desktop-only** gate |

### PR Validation (the broad, cheap gate)

Runs on `ubuntu-latest`: `bun install`, then **lint** (`oxlint`), **format check**
(`oxfmt --check .`), **build** of the `@<scope>/*` packages + web + api, **typecheck**, and the
**desktop unit suite** (`vitest`, ~600 tests). It validates the whole monorepo and does not
deploy. This is the check to keep fast and always-on.

### E2E Desktop (the narrow, expensive gate)

Runs the Playwright + Electron E2E suite for the desktop app on `ubuntu-latest`. It needs a
heavier environment than PR Validation:

- **`ffmpeg`** — `ffprobe` validates the exported media streams.
- **`playwright install-deps chromium`** — the system libraries Electron's Chromium links
  against (libnss3, libgbm, …). No browser is downloaded; the tests launch the app's own
  `electron` binary.
- **`xvfb`** — a virtual X display so the Electron window can open headless
  (`xvfb-run --auto-servernum bun run test:e2e`).
- **`--no-sandbox`** — added to the Electron launch args only under `CI`; the Chromium sandbox
  has no SUID helper on headless runners.

## Why two workflows and not one

This is the important part. The split is intentional and matches common industry practice.
The two are not "unit tests vs E2E" — they are **"cheap and universal" vs "expensive and
platform-specific"**, which is a natural seam.

1. **Different setup needs.** Only E2E needs `ffmpeg`, `xvfb`, and Chromium system libs.
   Keeping them apart keeps the always-on validation gate lean.
2. **Independent triggers.** E2E runs only when desktop code changes — it is `paths`-filtered
   to the desktop app dir, the workflow file, and `bun.lock` — so a docs typo or a web-only
   change never boots Electron + xvfb. This is only possible because it is its own workflow;
   folded into the universal gate, every change everywhere would pay the E2E cost. (For
   `pull_request`, GitHub evaluates the filter against the PR's whole diff, so a PR that touches
   the desktop still runs E2E on every push.)
3. **Clear, parallel signal.** They run concurrently and report as two checks. A flaky E2E
   run does not hide the lint/typecheck result, and vice versa.
4. **Independent required-status.** Branch protection is configured per check — you can make
   PR Validation required to merge while E2E stays non-required until it stabilizes (or the
   reverse). That needs two distinct checks.
5. **Independent evolution.** The E2E suite will grow (more flows, possibly a macOS job).
   Isolating it means changes there never risk the critical build/lint gate.

### The one cost, and its real fix

The only downside of separation is a duplicated `bun install` (each workflow checks out and
installs). **Merging does not fix this** — separate _jobs_ run on separate machines and do not
share a filesystem either, and collapsing into one _job_ loses the parallelism and the
per-workflow trigger. The right fix is **dependency caching** — both workflows cache bun's
global install cache (`~/.bun/install/cache`) via `actions/cache`, keyed on `bun.lock` — not
merging.

The design axis to keep straight:

- **Two workflows** (current) — most flexible: independent triggers, required-status, parallel.
- **One workflow, two jobs** — shared trigger, still parallel, but still no shared install and
  no easy per-job `paths` filter.
- **One workflow, one job** — shares install, but sequential and cannot path-filter a sub-step.

## The E2E harness (what E2E Desktop runs)

Playwright drives the **real production build** of the desktop app against a **fully isolated
data directory**.

- **Run locally:** `bun run test:e2e` from the desktop app dir (builds, then `playwright test`).
- **Isolation:** each run gets a throwaway `--user-data-dir` and a throwaway data vault seeded
  with a committed sample fixture, pointed at via a `preferences.json` — never the developer's
  real profile or documents.
- **Helpers:** a `launch.ts` helper (isolated launch + an `openEditor` that dismisses the
  onboarding overlay) and an `ffprobe.ts` helper (output validation).

### What it guards (both proven by red-green)

- **Playback / seek** — asserts the media protocol answers a `Range` request with `206` +
  `Content-Range` (a seek-wedge bug returned `200`). This assertion **also** guards the
  export `corsEnabled` fix, because the cross-origin `fetch` it uses requires `corsEnabled`.
- **Export** — runs the real export, waits for navigation to the saved output, and
  `ffprobe`-validates it (h264 video + aac audio, aligned durations).

### Two platform gotchas worth remembering

- **A small fixture cannot reproduce the seek-wedge via `<video>`.** A clip small enough to
  commit (~77 KB) buffers whole on first load, so a real seek stays in-buffer and never issues
  the `bytes=<offset>-` request that wedged Chromium. That is why the guard asserts the
  protocol's `Range` response **directly** (deterministic) rather than watching the `<video>`.
- **The export test skips on headless Linux CI.** Chromium there has no working WebCodecs
  **H.264 encoder**, so the export throws at runtime. `VideoEncoder.isConfigSupported`
  _falsely_ reports support on Linux, so the test gates on the environment instead —
  `test.skip(process.platform === "linux" && process.env.CI, …)` — a **visible** skip, never a
  silent pass. The export runs in full on macOS/dev machines. CI does not lose the root-cause
  coverage: the `corsEnabled` and `Range` fixes are still guarded on Linux by the playback
  Range-fetch test.

## Making a sometimes-skipped job a required check

> When only release candidates are verified and every other pull request should pass at
> once — stacked ones included — the full shape (classify job, gate that passes "not a
> candidate", no `branches:` filter) is in
> [release-gated-verification.md](./release-gated-verification.md#a-required-check-with-candidate-only-verification).

The gotcha above says a required check that never reports blocks merge forever. The same
bites a job that is present but **skipped by an `if:`** — and that is the normal shape of
an expensive job you only want on some PRs. You cannot require it directly: on every PR
where it is skipped, the merge is either blocked or (depending on how the skip is
reported) silently waved through, and neither is what you meant.

The pattern is a **gate job**: cheap, always runs, reads the expensive job's result and
reports for it. The gate is what you mark required.

```yaml
jobs:
  verify:                      # expensive; skipped on most PRs
    if: ${{ <the narrow condition> }}
    # …

  gate:
    name: Release candidate verified   # ← the required check
    runs-on: ubuntu-latest
    needs: [verify]
    if: always() && github.event_name == 'pull_request'
    steps:
      - env:
          APPLIES: ${{ <the narrow condition> }}
          RESULT: ${{ needs.verify.result }}
        run: |
          [ "$APPLIES" != "true" ] && { echo "Not applicable."; exit 0; }
          [ "$RESULT" = "success" ] && exit 0
          echo "::error::Not verified (verify job: $RESULT)."
          exit 1
```

`if: always()` is load-bearing: without it the gate is skipped whenever its dependency is,
which is the failure it exists to prevent. The cost is one billed minute per PR — GitHub
rounds a five-second job up to a whole minute — so count it before adding one to a
high-traffic repository.

### Draft → Ready for review as the trigger

Draft state is the readiness signal: a draft means "still accumulating, spend no runners",
and **Ready for review** means "verify this". GitHub already refuses to merge a draft, and
no bot churns the draft flag the way bots churn labels.

```yaml
on:
  pull_request:
    branches: [main]
    # ready_for_review   — THE trigger.
    # synchronize        — a new revision while ready; re-verify, cancel the superseded run.
    # converted_to_draft — back to accumulating; cancel in-flight verification.
    # opened/reopened    — a PR opened as ready needs its required check from the start.
    types: [opened, reopened, synchronize, ready_for_review, converted_to_draft]

jobs:
  verify:
    # Decided at JOB level, so a draft never boots a runner just to refuse.
    if: github.event.pull_request.draft == false
    concurrency:                       # on the expensive job only, never workflow-wide
      group: verify-pr-${{ github.event.pull_request.number }}
      cancel-in-progress: true

  cancel-superseded:
    if: github.event.action == 'converted_to_draft'
    concurrency:
      group: verify-pr-${{ github.event.pull_request.number }}
      cancel-in-progress: true
    runs-on: ubuntu-latest
    steps:
      - run: echo "Returned to draft; in-flight verification cancelled."
```

For a release PR, set `"draft-pull-request": true` in `release-please-config.json`. The bot
then rewrites a draft on every releasable merge and no rewrite allocates a runner; the owner
presses Ready for review when that candidate is the one to publish. Open ordinary PRs as
drafts too (`gh pr create --draft`) and mark them ready when they should be checked.

Three traps, each measured on a real release PR:

- **No `labeled` / `unlabeled`.** A label-based trigger listens to every label change, so a
  bot's own bookkeeping labels (`autorelease: pending`, `autorelease: tagged`) start full
  verifications — one of them 13 seconds after the PR had merged and the tag existed.
- **No `edited`.** The bot edits the PR body in the same second it force-pushes. The
  `edited` run verifies nothing and reports every check `skipped`, and GitHub counts a
  skipped required check as passing: the PR read "All checks have passed" while the real
  run was still working. Without `edited`, a stacked PR retargeted onto `main` needs one
  push, or close and reopen, to start its checks.
- **A commit's required check is the worst conclusion across every run of that name on
  that commit**, not the latest. `skipped` is neutral; a `failure` is permanent and a later
  success does not outvote it. So the gate reports from the run that verified the commit
  and stays silent on runs that did not; it must neither skip into a pass nor fail closed
  on a twin run.

Concurrency belongs on the expensive job, not the workflow: a workflow-level group lets an
irrelevant event cancel a valid verification before its own job conditions skip it.

The earlier version of this pattern used a `verify` label as both trigger and gate. Drop
it for the draft flag; the label is what produced the first trap above.

Forgetting to mark a PR ready must not be dangerous either. The release pipeline still has
its own `verify` (and platform E2E) behind `needs:`, so a candidate that merged unverified
fails at the tag, before anything is signed or published.

## Follow-ups

**Done:** the `paths:` filter on E2E Desktop and bun dependency caching in both workflows.

**Still open:**

- **A `macos-latest` job** to run the export test with a real H.264 encoder — the honest way to
  get full export coverage in CI, since macOS is the production platform.

  "If the minute budget allows" is not a plan, and macOS bills at ~10.3× Linux: one such job on
  every PR push has been measured consuming ~45% of an account's monthly allowance on its own.
  Pick its rung deliberately — see **[ci-runner-cost.md](./ci-runner-cost.md)** for the
  per-PR → on-merge → nightly ladder, the split-the-filter-with-the-job rule, and the guard that
  stops a platform-gated suite from passing silently on the wrong runner.

> **Gotcha — `paths` filters and _required_ checks.** A `paths`-filtered workflow that does not
> run reports **no status**, and GitHub treats a required-but-absent check as perpetually
> pending — which would **block merge** on any PR that doesn't touch the filtered paths. So a
> `paths`-filtered check must stay **non-required**, or you add an always-runs sentinel job
> (same check name) that trivially passes when the paths don't match.
