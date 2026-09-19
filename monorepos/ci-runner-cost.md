# CI runner cost

_Path filters decide **how often** a suite runs; the runner OS decides **what each run costs**.
Optimizing only the first leaves the larger half of the bill untouched — and the obvious fix for
the second can turn a suite green without testing anything._

## Problem

A personal account hit its GitHub Actions quota mid-month and every workflow began failing in
under five seconds with:

> The job was not started because recent account payments have failed or your spending limit
> needs to be increased.

The blocked job was a `release-please` run in a repository whose CI was already
[path-filtered and release-PR-skipped](./ci-per-project-pipelines.md) — textbook. It was blocked
by a **different repository's** spending.

Billing for the month, account-wide:

| SKU                  | Units       | Price/unit | Gross     |
| -------------------- | ----------- | ---------- | --------- |
| Actions Linux        | 1,144 min   | $0.006     | $6.86     |
| Actions macOS 3-core | 103.48 min  | $0.062     | $6.42     |
| Actions Windows      | 13 min      | $0.010     | $0.13     |
| Actions storage      | 0.18 GB-hr  | $0.000336  | <$0.01    |

Read that table twice. **103 minutes of macOS cost the same as 1,144 minutes of Linux.** One
`macos-latest` job, running per-PR-push in a single repository, consumed roughly **45% of the
account's entire monthly allowance** — while the carefully filtered Linux pipelines across five
repositories consumed the other half.

## The cost model

Three facts do most of the work.

**1. Runners are not priced alike.** Published rates:

| Runner            | Price/min | Multiplier vs Linux |
| ----------------- | --------- | ------------------- |
| Linux 2-core      | $0.006    | 1×                  |
| Windows 2-core    | $0.010    | ~1.67×              |
| macOS 3–4 core    | $0.062    | ~10.3×              |

A minute is not a minute. Included-minute quotas are consumed at the same multipliers, so
"2,000 free minutes" means 2,000 Linux minutes — or **194 macOS minutes**.

**2. The quota is per account, not per repository.** A noisy side project starves the release
pipeline of the product you actually ship. Cost is not a per-repo concern even when the workflow
is; the account is the blast radius.

**3. With no payment method, exhausting the quota blocks jobs — it does not bill you.**

> If your account does not have a valid payment method on file, usage is blocked once you use up
> your quota.

This is the difference between a surprise invoice and a release that cannot ship. It usually
surfaces at the worst moment, because the expensive workflows are the ones that exhaust it and
release workflows are the ones that then cannot run. Included allowances at the time of writing:
Free 2,000 min / 500 MB artifacts, Pro 3,000 / 1 GB, Team 3,000 / 2 GB.

**Set a non-zero spending limit regardless of plan.** Upgrading a plan raises the ceiling; only a
payment method removes the hard stop. Enable the 90% / 100% usage notifications too — the first
signal should not be a failed release.

## Two axes, not one

Cost is `runs × duration × multiplier`. Path filtering only attacks the first term.

| Axis                | Lever                                                    | Covered in                                        |
| ------------------- | -------------------------------------------------------- | ------------------------------------------------- |
| **How often**       | path filters, `paths-ignore`, release-PR skips, drafts   | [ci-per-project-pipelines.md](./ci-per-project-pipelines.md) |
| **What it costs**   | runner OS, job splitting, trigger tier                   | this document                                      |

A pipeline can score full marks on the first axis and still dominate the bill. Check the second
whenever a workflow names anything other than `ubuntu-latest`.

### The frequency trap that survives path filtering

`concurrency: cancel-in-progress` only kills runs that a **newer push supersedes while they are
still running**. A PR with four pushes spaced far enough apart pays for four complete runs. At 1×
nobody notices; at 10× that is a meaningful fraction of a monthly allowance for a single PR.

Shared-path filters compound it. `packages/**` and `bun.lock` sit in every per-project filter by
design — a shared-package change can break any consumer — which is correct for a Linux suite and
expensive for a macOS one. **Split the filter with the job:** the cheap job keeps the broad
filter, the expensive job watches only its own app.

## The escalation ladder

When an expensive suite costs more than its feedback is worth, move it down a rung rather than
deleting it. Each rung trades feedback latency for cost.

| Rung | Trigger                       | Feedback arrives      | Use when                                                        |
| ---- | ----------------------------- | --------------------- | --------------------------------------------------------------- |
| 1    | every PR push                 | before review         | Cheap runner, or the suite is the repo's main risk control.      |
| 2    | PR, narrow paths only         | before review         | Expensive runner, but the suite guards one app's own code.       |
| 3    | `push:` to `main`             | at merge              | Expensive runner; a post-merge revert is acceptable.             |
| 4    | nightly `schedule:`           | next morning          | Slow and broad; catches drift rather than a specific change.     |
| 5    | before release only           | at the tag            | The suite is really a release gate.                              |

Rungs 3 and below need a named owner for the failure. A red run on `main` that nobody watches is
not a safety net — it is a decoration. Wire a notification, or stay on rung 2.

Record the rung and its reasoning in the workflow header, with the condition that would move it
back up. A future reader needs to know the choice was deliberate and what would reverse it.

## The trap: a cheaper runner can make a suite pass without testing anything

The obvious optimization — "move this to `ubuntu-latest`, it is 10× cheaper" — is where real
coverage disappears, because **platform-gated tests skip rather than fail**:

```ts
test.skip(process.platform !== "darwin", "the local runtime is macOS-only in this plan");
```

On Linux that suite reports green. It ran, it passed, it asserted nothing. Nothing in the run
summary distinguishes it from a suite that genuinely verified the behaviour.

Before moving any suite to a cheaper runner, **grep it for platform gates**:

```bash
grep -rn "process.platform\|test.skip\|it.skipIf\|describe.skip" <suite-dir>
```

If the gate names the platform you are moving away from, the move deletes coverage. That includes
gates several layers down — a helper or fixture that no-ops off-platform is the same problem
wearing a disguise.

When a suite must run on one platform, make that a **failure**, not a skip. A one-line guard turns
an invisible loss into a red run:

```ts
test("this suite is running on the platform it asserts against", () => {
  expect(process.platform).toBe("darwin");
});
```

This is the same discipline [pr-checks.md](./pr-checks.md) applies to the export test's
environment gate — *a **visible** skip, never a silent pass*. The rule generalizes: any condition
that can silently reduce what a green check means should be loud somewhere.

## The trap: the expensive step may not be the only one that has to move

The natural split is "type-check and unit tests on Linux, platform tests on macOS". It often does
not pay what you expect, because **E2E runs against build output**:

```
test:e2e → needs out/ → needs build → must run on the same runner
```

So the build stays on the expensive runner too, and the saving shrinks to the unit tests alone.
Two ways out, neither free:

- **Upload the build as an artifact** from the cheap job and download it in the expensive one.
  Costs artifact storage and transfer time, and the artifact must be genuinely
  platform-independent — for Electron, native modules usually mean it is not.
- **Accept the duplicate build.** Simple, and often correct: the cheap job's build is still
  earning its keep as a type-check and bundle-integrity check for everything the expensive job
  does not trigger on.

Measure the steps before assuming the split pays. If the expensive job is dominated by install and
build rather than by the tests, splitting buys very little and the **trigger tier** is the real
lever.

## Measure before optimizing

Reading the REST API consumes **no Actions minutes** — the two are separate products. This is
worth saying out loud because it is exactly the moment when the instinct is to touch nothing: the
measurement is free even while the account is blocked.

```bash
gh api --paginate "/repos/{owner}/{repo}/actions/runs?per_page=100"   # runs, conclusions, triggers
gh api "/repos/{owner}/{repo}/actions/runs/{run_id}/timing"           # billable ms, per OS
```

The four numbers that decide the fix:

1. **Runs per month per workflow.** Many short runs and few long ones need opposite fixes.
2. **Billable minutes by OS.** The only figure comparable to the bill; raw duration is not.
3. **Runs superseded on the same PR.** High means the frequency is the problem, not the job.
4. **Runs triggered by a shared path** (`packages/**`, `bun.lock`) **rather than the app's own
   code.** High means the filter is the problem, not the trigger tier.

Then compare against the repository's share of the account bill — GitHub's billing page breaks
usage down by repository, which is the fastest way to find which repo to look at first.

## Artifacts and storage

Storage is billed by GB-hour, so **retention is a multiplier on size**. A 500 MB desktop
installer kept for 90 days is not the same line item as the same file kept for 3.

- Set `retention-days` explicitly on anything large; the default is 90.
- Do not upload an artifact a release already publishes elsewhere. Gate it —
  `if: ${{ !startsWith(github.ref, 'refs/tags/') }}` keeps the convenience copy for manual runs
  without duplicating every tagged release into Actions storage.
- Free's 500 MB allowance is smaller than a single Electron build's output. If a workflow uploads
  installers, storage is a real constraint, not a rounding error.

## Checklist

Before merging a workflow that names a non-Linux runner:

- [ ] Does this need the expensive runner, or only the tests inside it do?
- [ ] Which rung of the ladder is it on, and is that written down in the header?
- [ ] If it is on rung 1–2, does its path filter watch only its own app?
- [ ] If anything moved to a cheaper runner: grep for platform gates, and add the guard test.
- [ ] Are large artifacts given an explicit `retention-days`?
- [ ] Is a non-zero spending limit set, with usage notifications on?

## Related

- [ci-per-project-pipelines.md](./ci-per-project-pipelines.md) — the frequency axis: universal
  fast check plus path-filtered per-project suites, and why docs-only PRs still run the formatter.
- [pr-checks.md](./pr-checks.md) — PR gate shapes, the visible-skip discipline, and the
  `paths`-filter/required-check gotcha.
- [testing/ci.md](./testing/ci.md) — running the suites themselves in CI.
