# Design workflow: from idea to a spec + ADR PR

> **One-minute summary.** A feature moves through one lifecycle with one artifact per stage:
> a captured idea (backlog item) → a roadmap promise → design (research, an ADR-contradiction
> check, options in prose, a hard gate) → a spec with zero or more ADRs, shipped in its own PR →
> an ephemeral plan on the code branch, deleted when the feature ships. Nothing is "updated
> later" — each stage has one owner and one moment it happens in.

The executable version of this convention is the `cs-design` skill (**planned** — not built
yet; until it exists, use `superpowers:brainstorming` and `superpowers:writing-plans`). This
page is the reasoning; the skill is the procedure.

## The lifecycle

```
idea mid-task ──capture──▶ backlog item                                   status: promised
                                │  "let's build this"
                                ▼
              roadmap: phase + one line → item                (edited now, never at ship)
                                │
      design ── research · ADR-contradiction check · options in prose
                 → one reply → HARD GATE
                                ▼
              PR "design: <slug>" — spec + 0..n ADRs        references the item   status: designing
                                │  merged
                                ▼
              branch feat/<slug>: plan (ephemeral) → execution                    status: building
                                │
              shipping PR: feature doc updated · plan deleted · item closed
                                ▼  merged
              item closed (completed)                                            status: shipped
```

| Piece | Question it answers | Lifetime | Created by | Retired |
| --- | --- | --- | --- | --- |
| Backlog item | What did I notice and want built or redone? | Until closed | capture step | Closed by the shipping PR, or closed not-planned with a reason |
| Roadmap line | What has this product promised, in what phase? | Product lifetime | promote step | Never deleted; pruned when its item closes |
| Spec | What, why, decided, open | Permanent record | design | Never |
| ADR | Why this and not that | Permanent | design, 0..n per spec | Superseded, never deleted — see [adr-pattern.md](./adr-pattern.md) |
| Plan | In what order | Code branch only | planning step, after the spec merges | Deleted by the shipping PR |
| Feature doc | How it works today, and why | With the code | ship step | When the feature is removed — see [feature-docs-as-memory.md](./feature-docs-as-memory.md) |

Backlog items and the roadmap's promise-only shape, with derived status, are
[backlog-and-roadmap.md](./backlog-and-roadmap.md)'s rule; this page starts where an item is
picked up for design.

## Design procedure

1. **Classify, and only ratchet upward.** Spike (a question and a probe; nothing written) |
   bounded (a short design discussed in chat, then a gate, then implementation; an ADR only if
   step 2 below says so) | architectural (full procedure). Announce the classification before
   proceeding.
2. **Read before deciding — stop at the first rung that answers:**
   - does this need to exist at all?
   - is it already decided? (check the ADR log, the roadmap, feature docs, hub pages linked
     from the repo's own instructions file, and the backlog item's thread)
   - **does it contradict an existing ADR?** Finish the design anyway, then name the
     contradiction and that ADR's reopen condition explicitly in the spec — never decide around
     it silently.
   - does an existing codebase pattern already cover it? Reuse it.
   - otherwise, decide.
3. **Input**: the backlog item's body plus anything pasted into the conversation. On an
   explicit "read this and keep it" request: general, reusable content goes straight to the
   hub via `cs-update-knowledge-hub` (the project links it, never duplicates it); anything
   specific to this project goes into the backlog item and, once the spec is approved, into
   the spec's own "Sources and unverified claims" section. There is no separate
   project-local knowledge folder for this — one home per fact, not two.
4. **Research the state of the art, always.** Cite what was read, even for a bounded change.
5. **Options in prose**: numbered, each with its implications and the strongest argument
   against it, ending in a recommendation. Never a multiple-choice picker — the reader answers
   in one free-text reply that can mix parts of several options, and that reply is folded in.
6. **Hard gate.** No code, no scaffolding, no dependency installs before the spec (and its
   ADRs, if any) is approved.
7. **Write the spec and its ADRs**, self-review for placeholders, contradictions, scope creep
   and ambiguity, and open them as one PR titled `design: <slug>` that references the backlog
   item. One review decides it.
8. **On merge**: report the next step — planning on a `feat/<slug>` branch, plan file deleted
   when that feature ships.

## The spec

A spec opens with the same one-minute summary block every page here does — the decider reads
that block, not the whole document, most of the time. It carries:

- **What, why, decided, open** — the decision record.
- **Zero, one, or several ADRs.** A spec that only reuses decisions already on record needs
  no new ADR at all; see [adr-pattern.md](./adr-pattern.md) for when one is warranted.
- **A testing section naming the failures this change must survive** — the input a test-audit
  step later turns into one owner test per failure. This is spec-level, not plan-level: it is
  written once, while the decisions are fresh, and every task later drawn from the spec can
  cite it instead of re-deriving it.
- **A "Sources and unverified claims" section** for anything project-specific that came out of
  a "read this and keep it" request but isn't general enough for the hub.

The spec and its ADRs merge in their own PR, separate from the code PR that follows. Reviewing
a design on its own — before any code exists to distract from it — is what makes the review
useful, and it is what makes "designing" and "building" distinguishable as a derived status
(see [backlog-and-roadmap.md](./backlog-and-roadmap.md)).

## The plan is ephemeral

The plan is written after the spec merges, lives on the feature's code branch only, and is
never reopened once written — a decider who wants to know why reads the spec and the ADRs, not
the plan. The shipping PR deletes it in the same commit that finishes the feature; its last
commit stays reachable through `git show`, and the shipping PR body links it. This is the
ship step of the handoff convention, [verifiable-handoffs.md](./verifiable-handoffs.md) —
**planned**: today that convention does not yet describe deleting the plan or updating the
feature doc; both are meant to join it.

## Related

- [adr-pattern.md](./adr-pattern.md) — the format a spec's decisions become, and when a
  decision needs one at all.
- [backlog-and-roadmap.md](./backlog-and-roadmap.md) — where a design's input comes from, and
  the derived status this workflow produces.
- [feature-docs-as-memory.md](./feature-docs-as-memory.md) — what a shipped design leaves
  behind for the next agent to read instead of the spec.
- [verifiable-handoffs.md](./verifiable-handoffs.md) — the ship step that closes a design's
  lifecycle.
- [lesson-distillation.md](./lesson-distillation.md) — a lesson that becomes a new skill still
  starts here, at design.
