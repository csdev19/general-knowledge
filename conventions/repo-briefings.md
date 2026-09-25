# Repo briefings

**A repository should be able to explain itself in one read, to each kind of
reader, from its own truth.** A pitch for a person, a context briefing for an
AI, a stack rundown for an evaluating developer, a roadmap for "where is
this?". Split by audience, not by topic: the same fact reads differently to
each.

The executable version of this convention is the
[`generate-briefings` skill](https://github.com/niway-dev/skills/tree/main/skills/generate-briefings).
This page is the reasoning; the skill is the procedure.

## The rule

A briefing is regenerated from repo truth, never from memory and never from
plan documents. Plans record intent and lag reality; a briefing that quotes
one is describing what was meant, not what shipped.

Every briefing carries a date stamp, and the stamp is a promise: everything
under it was checked that day. Bumping the date without re-verifying breaks
the promise silently, which is worse than an old date.

## Why a binding file

Every repo puts its docs somewhere different, names them differently, and
has its own language and validation rules. The skill does not guess: a small
binding file in the repo says where the briefings live, which exist, what the
sources of truth are, and how to validate. The binding wins on *where and
how*; the briefing contracts win on *what each must contain*.

## Three modes, one distinction

- **Instantiate**: the repo has no binding; interview the owner, then write.
- **Refresh**: regenerate from sources and update the stamps.
- **Audit**: compare stamps against the last commit touching each source,
  spot-check the claims most likely to have moved, report. **Writes
  nothing.** "Fix this one date" during an audit is a refresh the user did
  not ask for.

## Shipped means merged

Roadmap status comes from merged PRs or, without GitHub, from the log since
the last tag, and the briefing says which. A truncated PR list is never
presented as complete. The "next" bucket has no automatic source: it comes
from the owner, never promoted from a plan.

## Related

- [verifiable-handoffs](./verifiable-handoffs.md): a briefing refresh is
  handed back like any other work.
- [specs-and-plans-workflow](./specs-and-plans-workflow.md): the plans a
  briefing must not quote as status.
