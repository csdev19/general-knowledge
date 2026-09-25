# Audit briefs

**Work that someone else must check is briefed so they can attack it, not
admire it.** A fresh-context agent or reviewer with no memory of the session
needs the intent, the evidence, and the writer's own weakest points, or they
will audit the wrong thing.

The executable version of this convention is the
[`cs-write-audit-brief` skill](https://github.com/niway-dev/skills/tree/main/skills/cs-write-audit-brief).
This page is the reasoning; the skill is the procedure. Its sibling,
[verifiable-handoffs](./verifiable-handoffs.md), is for a **person deciding
whether to trust finished work**; an audit brief is for an **agent sent to
find what is wrong**, including mid-task.

## The four failures it prevents

| Failure | What it looks like |
| --- | --- |
| Intent lost | The auditor sees the diff, not the request, and reviews code as written instead of against the ask |
| Claims and evidence blurred | "Verified" and "assumed" read the same, so the auditor re-checks the solid and trusts the shaky |
| Settled decisions re-litigated | A choice the user already made comes back as a finding |
| A green check that proved nothing | A tool ran, exited clean, and covered none of the work |

## Three rules that carry the value

**Every claim carries how it is known.** Verified this session, with the
proof; carried over from a ticket or prior session, unchecked; or a judgement
call, with the alternative rejected. Nothing in between.

**Every decision carries an owner and a status.** A decision the user made
and settled is out of bounds for the auditor: flag a consequence, never
reverse it. A decision the agent made alone is exactly where to attack.

**A check reports what it covered, not only its exit code.** A docs-only
change in a task-graph monorepo, a suite with no case touching the changed
path, a linter that skips this file type: each exits 0 having validated
nothing.

## "Where to attack" is never empty

The writer lists their own weakest points, worst consequence first. If nothing
comes to mind, the claims were not sorted honestly: an agent's unilateral
calls, carried-over claims, and anything delivered beyond the ask are always
candidates. The list is a floor for the auditor, not a ceiling.

## Kept live, in one place

The brief is regenerated at the same path whenever the work moves, never
appended. An auditor reading an older state produces confidently wrong
findings.

## Related

- [ai-agent-delegation](./ai-agent-delegation.md): which model the auditor
  runs on, and why that is announced before dispatch.
