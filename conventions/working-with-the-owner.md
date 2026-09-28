# Working with the owner

> **One-minute summary.** The owner thinks by dropping a large block of context at once and
> expects it processed into durable knowledge, not stored verbatim; argued with, not agreed
> with by default; and returned as prose options with a recommendation, answered in one
> free-text reply rather than a multiple-choice picker. Approval is coarse-grained — the idea,
> then the whole document, not section by section. Decisions get written down with their
> reopen condition. Nothing grows without a reason.

This page is working-style guidance for whoever — human or agent — works with the hub's
owner, distilled from how design conversations with him actually went. It carries no personal
data beyond that working style.

## How he thinks and decides

| Trait | What it means in practice |
| --- | --- |
| Dumps context in one large message, then expects synthesis. | Do not ask for the context piecemeal; process what arrives in one pass. |
| Wants the dump **processed, not stored**. | Simplify, drop redundancy, keep intent. Ask when intent is unclear rather than guessing. |
| Wants to be asked about his own gaps. | Surface logic holes, unverified claims and missing information in his own input — do not silently patch over them. |
| Wants to be argued with. | Give the strongest counterargument to his own proposals, so he can adopt a suggestion partially instead of only accept-or-reject. |
| Decides from evidence and the state of the art, not the first proposal. | Research always precedes a recommendation; if the community has settled something and supports it, that carries real weight. |
| Answers in one free-text reply that mixes options. | Never present a single-choice picker for a design decision; prose options he can respond to in one message, mixing parts of several, are what work. |
| Approves at coarse grain. | The general idea once, then the finished document once — not section by section. |
| Prefers a practical summary with references over long documents to read cover to cover. | Lead with the one-minute version; put the detail behind links for whoever wants to check it. |
| Removes what does not serve; keeps what works. | Default to not touching what isn't broken, but actively cut what has stopped earning its place. |
| Willing to take the heavier, more correct option when the direction is right. | Given a real trade-off between "faster" and "right the first time," he leans toward doing it right, including paying for a more capable model to get a design right the first time. |
| Distrusts growth without a reason. | New repos, new CI jobs, new docs pages each need a reason stated, not accumulated by default. |

## What he wants written down, and where

| Rule | Why it matters |
| --- | --- |
| Reusable knowledge lives in the hub; a project links to it and records only its own application. | See [knowledge-hub-maintenance.md](./knowledge-hub-maintenance.md). |
| Knowledge acquired in a session should become a skill (the procedure) paired with a hub page (the reasoning). | See [lesson-distillation.md](./lesson-distillation.md). |
| A context dump becomes a provisional working document first, from which specs and ADRs are drawn later — not documentation in its own right. | The dump is an input, not a deliverable. |
| An ADR records context, decision, rejected alternatives, consequences, explicit non-goals, and a reopen condition. | See [adr-pattern.md](./adr-pattern.md); applied consistently across his own products. |
| The pieces he actually wants to work with are backlog, roadmap, spec, ADR, and feature doc. | A separate, standalone implementation plan is not one of them — it is ephemeral by design. See [design-workflow.md](./design-workflow.md). |
| A spec can carry zero, one, or several ADRs; a feature may only reuse a decision already on record. | See [adr-pattern.md](./adr-pattern.md), "when an ADR is needed." |
| Before deciding, existing decisions get checked for contradiction, not just precedent. | The read-before-deciding ladder in [design-workflow.md](./design-workflow.md). |
| Everything published is English; conversation may be in another language. | See [lesson-distillation.md](./lesson-distillation.md), "Company work stays with the company," for the related placeholder rule. |
| Documentation is honest about what is and is not implemented. | A design draft says plainly which of its own assumptions turned out wrong, rather than hiding them. |

## How he wants to be worked with

| Rule | Why |
| --- | --- |
| Name the model before dispatching a subagent. | Design work runs on a more capable model chosen explicitly; execution runs on a cheaper one by default. He should never have to guess what ran or on what. |
| Hold a hard gate: no code before the design is approved. | This is the part of any planning process he values most and keeps deliberately. |
| Batch the work, then report exactly where each change landed. | "Generate everything at once, then tell me the links" beats a narrated, incremental report. |
| Land work fully: commit, push, open the PR — never leave it stranded in an isolated workspace. | See [verifiable-handoffs.md](./verifiable-handoffs.md). |
| Company or client information never enters his personal repositories; placeholders unless he names it, and he approves the final text before it ships. | See [lesson-distillation.md](./lesson-distillation.md), "Company work stays with the company." |
| Do not over-engineer the process; apply the agreed approach and stop. | Once a playbook is chosen, the goal is applying it, not elaborating it further. |

## Testing preferences

Already captured as its own convention, not repeated here: a test earns its place only by
catching a credible failure nothing else catches, and the failure list comes before the test.
See [test-audits.md](./test-audits.md).

## Related

- [design-workflow.md](./design-workflow.md) — the procedure this working style is designed
  around: research, prose options, one reply, a hard gate.
- [adr-pattern.md](./adr-pattern.md) — the decision format he applies consistently.
- [lesson-distillation.md](./lesson-distillation.md) — how a session's lesson finds its one
  home, and the placeholder rule for company work.
- [verifiable-handoffs.md](./verifiable-handoffs.md) — what "land the work" means in practice.
