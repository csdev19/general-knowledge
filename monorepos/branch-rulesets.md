# Branch rulesets: making a check actually required

The hub's cost pages keep saying "make `verify` required on `main`". This page is the
mechanism, and the two ways turning it on breaks things that were working a minute ago.

Rulesets are the current API. Classic **branch protection**
(`repos/{o}/{r}/branches/main/protection`) still works and still appears in older docs, but
rulesets are what the UI writes now, they compose (several can apply to one branch), and
they are the only one of the two with a sane JSON shape for scripting. A repository can
have zero rulesets and still be unprotected in a way that is easy to miss — see the audit
at the bottom.

## The shape

Three rules cover the trunk of a single-owner repository:

| Rule                     | What it stops                                             |
| ------------------------ | --------------------------------------------------------- |
| `deletion`               | Deleting `main`.                                          |
| `non_fast_forward`       | Force-pushing over `main`.                                |
| `required_status_checks` | Merging while the named check is missing, pending or red. |

```bash
gh api -X POST repos/<owner>/<repo>/rulesets --input ruleset.json
```

```json
{
  "name": "main release",
  "target": "branch",
  "enforcement": "active",
  "conditions": { "ref_name": { "include": ["~DEFAULT_BRANCH"], "exclude": [] } },
  "rules": [
    { "type": "deletion" },
    { "type": "non_fast_forward" },
    {
      "type": "required_status_checks",
      "parameters": {
        "strict_required_status_checks_policy": false,
        "do_not_enforce_on_create": false,
        "required_status_checks": [
          { "context": "Release candidate verified", "integration_id": 15368 }
        ]
      }
    }
  ],
  "bypass_actors": []
}
```

Four fields in there are decisions, not boilerplate:

- **`~DEFAULT_BRANCH`** rather than the literal `main`. The rule follows a rename.
- **`integration_id: 15368`** pins the check to the GitHub Actions app. Without it the
  context matches on **name alone**, so any app — or a human with a token hitting the
  statuses API — can satisfy the rule by reporting a green check called
  `Release candidate verified`. Omitting it is the difference between "Actions verified
  this" and "something claimed it did".
- **`strict_required_status_checks_policy: false`** means the PR does not have to be
  up to date with `main` before merging. Setting it `true` re-runs CI on every PR each
  time anything lands on the trunk, which on a busy repo is the single most expensive
  toggle on this page.
- **`bypass_actors: []`**. An empty list means nobody bypasses, including the owner
  (`current_user_can_bypass: "never"`). Add actors deliberately or not at all.

Update with `-X PUT .../rulesets/<id>` and the same body. Reverse with
`-X DELETE .../rulesets/<id>` — instant, and it unblocks everything at once, which makes
this a cheap thing to try.

## Trap 1: the required check must be one that ALWAYS reports

This is the same trap as the `paths` filter and the skipped job, in a third disguise, so
it is worth stating as one rule:

> **A required check that does not report is not "skipped" — it is pending, forever.**

GitHub has no way to distinguish "this check will never come" from "this check has not
come yet", so it waits, and the merge button stays grey. Three ways to walk into it:

1. Require a `paths`-filtered workflow. Any PR outside those paths blocks.
2. Require a job that an `if:` skips. Same outcome on every PR where it skips.
3. Require a check on a repository whose open PRs predate it — Trap 2 below.

The fix for 1 and 2 is the gate-job pattern in
[pr-checks.md](./pr-checks.md#making-a-sometimes-skipped-job-a-required-check): a cheap job
with `if: always()` that reads the expensive job's result and reports for it. **Require the
gate, never the expensive job.**

## Trap 2: turning it on blocks every open PR, retroactively

A status check is not a property of a pull request. It is a result attached to a **commit**,
produced by a run that happened at some point in time.

So when you add a required check to a repository that already has open PRs, GitHub looks at
each PR's head commit for a check of that name — and on any PR whose head was pushed
**before the workflow producing that check existed**, there is nothing there. Never ran,
never will. Those PRs are blocked, showing
_Expected — Waiting for status to be reported_, until something moves their head.

Measured on a real repository: four open PRs, heads from 4 to 15 days old, all four blocked
the moment the ruleset went active, on a check whose workflow had merged an hour earlier.

Check before you enable, not after:

```bash
# which open PRs already carry the check you are about to require?
gh pr list --state open --json number,headRefOid,statusCheckRollup \
  --jq '.[] | "\(.number)  \(.headRefOid[0:8])  \([.statusCheckRollup[]?.name] | join(", "))"'
```

The remedy is one push per PR — a rebase onto the trunk, a real commit, or
`git commit --allow-empty -m "chore: trigger checks" && git push`. Anything that moves the
head starts a run, the run reports, the PR unblocks. It is trivial; the point is to know it
before the owner finds four dead PRs.

Enable the ruleset **after** the workflow producing the check is on the trunk and has
reported at least once, and say in the announcement that open PRs need a push.

## Trap 3: a stacked PR is not protected, and has no checks at all

A branch ruleset on `~DEFAULT_BRANCH` governs pull requests **targeting** the default
branch. A stacked PR targets the branch below it in the stack, so the ruleset does not apply
to it.

Worse, and independently: in a `pull_request` trigger, `branches:` filters the **base** — the
branch the PR wants to merge into — not the head it comes from. This reads backwards to most
people.

```yaml
on:
  pull_request:
    branches:
      - main # ← the TARGET of the PR, not the PR's own branch
```

So in a four-PR stack where only the bottom one targets the trunk, the other three match no
trigger and **no workflow runs on them at all**. Not red. Not pending. Absent: their
`statusCheckRollup` is an empty list.

```bash
gh pr view <n> --json statusCheckRollup --jq '.statusCheckRollup | length'   # 0
```

That empty state renders as an untroubled PR page, which is exactly how it gets read as
"green". It is not green; nobody looked.

This is a reasonable trade-off — a stack is reviewed and merged bottom-up, and the bottom PR
does face the trunk's rules — but two habits follow from it:

- **Never cite CI as evidence on a stacked PR.** Say plainly that it is unverified by CI and
  that the evidence is the local hook plus review.
- If a middle PR genuinely needs a cloud run, retarget it at the trunk or dispatch the
  workflow by hand; do not add a trigger for feature-branch bases, which would multiply runs
  across the whole stack.

## Auditing a repository

```bash
R=<owner>/<repo>

# Rulesets. An empty array means nothing is enforced, whatever the UI suggests.
gh api "repos/$R/rulesets" --jq '.[] | "\(.id)  \(.name)  \(.target)  \(.enforcement)"'
gh api "repos/$R/rulesets/<id>" --jq '{rules, bypass_actors, current_user_can_bypass}'

# Classic branch protection, separately — a repo can have one, the other, both or neither.
gh api "repos/$R/branches/main/protection" 2>/dev/null || echo "no classic protection"
```

Check both. "No rulesets" and "not protected" are two different reads of the same repository
and neither implies the other.

## Checklist

- [ ] The required check is a job that runs on **every** PR to the trunk, whatever it touches.
- [ ] `integration_id` pins it to the app expected to report it.
- [ ] `bypass_actors` is empty, or every entry is deliberate.
- [ ] `strict_required_status_checks_policy` is `false` unless the re-run cost was measured.
- [ ] The producing workflow is on the trunk and has reported once, **before** enabling.
- [ ] Open PRs were listed first, and their owners told they need one push.
- [ ] Nobody reads a stacked PR's empty check list as a pass.

## Related

- [pr-checks.md](./pr-checks.md) — the gate-job pattern, and the label-as-trigger pattern.
- [release-gated-verification.md](./release-gated-verification.md) — what the required check
  should be verifying, and why the release boundary is the enforcement point.
- [ci-runner-cost.md](./ci-runner-cost.md) — every gate job bills a whole minute; count them.
