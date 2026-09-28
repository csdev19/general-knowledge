# ADR pattern

> **One-minute summary.** An ADR records context, decision, rejected alternatives,
> consequences, explicit non-goals, and the condition that would reopen it — never just the
> decision. Write one when a choice is hard to reverse, contested, or the kind of thing a
> future reader will ask "why didn't we just—"; skip one when the change only reuses a
> decision already on record, or is reversible enough that a spec section covers it. An ADR is
> superseded or amended; it is never deleted or silently rewritten.

## Format

```markdown
# ADR <NNNN>: <short decision title>

- **Status:** <Proposed | Accepted | Superseded | Amended>
- **Date:** YYYY-MM-DD
- **Supersedes / Superseded by / Amends:** <ADR number, or N/A>
- **Implementation:** <where the decision landed, or "scaffold only" while partial>

## Context

The situation and constraints that made this a decision at all — including what was tried
before and what specifically stopped working.

## Decision

The choice, stated as an instruction, with the concrete pieces that make it checkable (pinned
versions, a naming rule, a boundary), not just the intent.

## Alternatives rejected

Each alternative considered, in one line naming what it would have given up.

## Consequences

What changes because of this decision, including costs accepted knowingly, not only benefits.

## Non-goals

What this decision explicitly does not cover or promise, so a later reader doesn't read one
in that isn't there.

## Reopen when

The observable condition that would bring this decision back up for review — not "if it turns
out to be wrong," but the specific signal that would mean that.
```

An ADR that records an empirically measured choice can add a **Measured** section between
Alternatives rejected and Consequences: the numbers, the method, and — this matters more than
the numbers — what the measurement does *not* cover yet. A decision only partly confirmed by
its own measurement says so there, rather than rounding up.

## When an ADR is needed, and when it is not

An ADR is warranted when the decision is:

- hard or costly to reverse (a dependency choice, a data shape, a naming rule other things
  key off of);
- contested or non-obvious enough that a future reader would otherwise re-litigate it;
- the kind of choice [design-workflow.md](./design-workflow.md)'s classification step ranks
  **architectural**, or a **bounded** change whose read-before-deciding ladder found it
  contradicts an existing ADR (in which case: finish the design, then write the new ADR and
  name the contradiction and the old ADR's reopen condition explicitly — never decide around
  it silently).

No new ADR is needed when:

- a **spike** classification never reaches a design step at all;
- the change only **reuses a decision already on record** — a feature that fits inside an
  existing ADR's decision needs no ADR of its own, only a link to it;
- the choice is a **reversible default** small enough that a line in the spec's own decision
  table covers it.

## How it links to the spec and the feature doc

A spec carries zero, one, or several ADRs, created in the same `design: <slug>` PR
([design-workflow.md](./design-workflow.md)). The spec is the record of what was decided and
why *for this feature*; an ADR is the record of a reusable decision that outlives the feature
that produced it — a spec can retire, an ADR keeps being cited by whatever reuses its
decision.

A feature doc's "Why it is this way" section is a table of `Decision → ADR / spec` links,
never restated reasoning ([feature-docs-as-memory.md](./feature-docs-as-memory.md)). The ADR
stays the one place the reasoning lives; every other document points at it.

## Measured example

ADR-0022 in one of the owner's own products records a checkpoint upgrade for a speech
transcription engine. Its **Measured** section reports a benchmark across three engines and
two languages, states plainly that the new checkpoint regresses on one metric against its
predecessor, and reopens — and immediately re-decides — a related ADR the same day, on the
evidence the same measurement produced. Its **Reopen when** section separates a condition
already triggered (and resolved in the same sitting) from one still open, rather than leaving
"reopen when" as a single unfalsifiable line.

## Related

- [design-workflow.md](./design-workflow.md) — the lifecycle stage that produces an ADR, and
  the ladder that decides whether one is needed.
- [feature-docs-as-memory.md](./feature-docs-as-memory.md) — where an ADR is cited once a
  feature ships, never copied.
- [knowledge-hub-maintenance.md](./knowledge-hub-maintenance.md) — an ADR that records
  reusable, product-agnostic knowledge belongs here in the hub, not only in the project.
