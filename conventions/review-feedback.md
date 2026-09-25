# Review feedback

**A review comment is a claim to verify, not an instruction to execute.**
Reviewers are often right, sometimes wrong, and occasionally asking for
something the repository has already decided against. Treating every comment
as a task produces performative agreement, reverted decisions and code that
is worse after the review than before it.

The executable version of this convention is the
[`cs-address-review-feedback` skill](https://github.com/niway-dev/skills/tree/main/skills/cs-address-review-feedback). This page
is the reasoning; the skill is the procedure.

## The rule

Every comment is classified **after reading the code at the line**, never
from the comment alone, into one of six kinds, and each kind has one
treatment:

| Kind | Treatment |
| --- | --- |
| Correct | Reproduce first (a failing test, a command), then fix |
| Preference | Apply; it is the reviewer's codebase too |
| Against convention | Decline, quoting the rule's location |
| Settled decision | Decline in the thread, escalate to the owner |
| Question | Answer in the thread |
| Wrong | Show why, with the line or the output; no fix |

When the kind depends on a fact nobody has established, the comment is a
question back to the reviewer and the code does not move. A boundary claim
("should be `>=`") is the common case: whether the edge is in or out is a
specification, and "the reviewer said so" is not one.

## Why decisions are not reopened in a PR thread

An ADR or a dated decision exists so the question stays closed. A reviewer
asking to change it is not wrong to ask, but the PR author is the wrong
person to answer: the change belongs to the decision's owner, through the
record that made it. The thread gets the pointer; the owner gets the
escalation.

## Why the reply cites a commit

"Done" cannot be checked. "Fixed in `abc1234`: the boundary is now
inclusive" can. That means the push comes before the reply, and each
accepted comment gets its own named commit, so the reviewer can look at
exactly the change they asked for and nothing else.

## Why threads are left open

The reviewer opened the thread to check something. They close it when they
have. Resolving it for them removes the one signal that tells them what is
still pending.

## Related

- [verifiable-handoffs](./verifiable-handoffs.md): a fix that changes
  behaviour updates the PR body under the same rules.
- [architecture/decisions](../architecture/decisions/): the records a
  settled-decision comment points at.
