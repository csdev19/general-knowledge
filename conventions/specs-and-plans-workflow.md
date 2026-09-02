# Specs and Plans Workflow

_The brainstorm → spec (design) → plan (implementation) → archive flow, and how it connects to the backlog._

Two sibling folders hold all the work with substance:

```
docs/
├── specs/    # Design documents: WHAT and WHY
└── plans/    # Implementation plans: HOW, step by step
```

Separating design from implementation avoids mixing "what we want and why" with "in what order we touch the files". A spec can be discussed and approved before a single line of plan is written.

## The flow

```
brainstorm  →  spec (design)  →  plan (implementation)  →  ship  →  archive
   (idea)      specs/YYYY-...     plans/YYYY-...          (PRs)   (move the learning)
```

1. **Brainstorm** — the idea is born, usually as a ⚪/🔵 item in the [backlog](./backlog-pattern.md).
2. **Spec (design)** — once the idea is worth it, a design doc goes into `specs/`. It defines the goal, the non-goals, the architecture and the decisions. This is what gets reviewed and approved.
3. **Plan (implementation)** — from the approved spec, a plan goes into `plans/`: concrete tasks, order, files to create/touch, checkboxes to track progress. A large spec can be split into several numbered plans.
4. **Ship** — the plan is executed task by task (code + tests + docs). The checkboxes mark progress. **To execute with agents in parallel**, the plan is first converted into self-sufficient backlog deliverables and becomes superseded — see [plan → backlog](./plan-to-backlog.md).
5. **Archive** — once shipped: the durable learning moves into the feature/architecture doc, the item leaves the backlog map, and the spec/plan remain as a historical record.

## Naming convention

Files with the date up front, for a natural chronological order:

```
specs/YYYY-MM-DD-<topic>-design.md      # e.g. 2026-06-28-version-gate-design.md
plans/YYYY-MM-DD-<topic>.md             # e.g. 2026-06-28-version-gate.md
```

- The **spec** carries the `-design` suffix.
- The **plan** shares the date and topic with its spec, without the `-design`.
- When a plan is split into phases, they are numbered: `...-video-editor-01-foundation.md`, `...-02-timeline.md`, etc.
- The date is the doc's creation date, not the ship date.

## What goes in each

### Spec (`specs/`) — design

- **Header**: title, date, branch, scope (which apps/packages it touches).
- **Goal / Non-Goals**: what it solves and, explicitly, what is left out (deferred).
- **Architecture**: the approach, the data flow, the key decisions and their rationale.
- **Files**: what gets created and what gets modified (at a high level).

It answers **WHAT** and **WHY**. It does not get into the tactical order of the implementation.

### Plan (`plans/`) — implementation

- **Goal / Architecture / Tech Stack**: an executable summary, one or two lines each.
- **Reference to the spec**: a link to the design doc it comes from.
- **Conventions to read before starting**: the repo's language, commit rules, how to run tests, etc.
- **File structure**: create/modify, explicitly.
- **Tasks with checkboxes** (`- [ ]`): in order, ideally TDD-style, to track progress and allow agentic execution.

It answers **HOW** and **IN WHAT ORDER**.

## How it connects to the backlog

- The **backlog** is the map of "what's coming"; `specs/` and `plans/` are the detailed work behind an item with substance.
- A backlog item that gets started links to its spec and its plan (`Design: /specs/...` · `Plan: /plans/...`).
- On ship, the item is marked 🟢/✅ and leaves the map; the spec/plan stay archived as a record.

## Why it helps

- **Design before code**: the spec is approved without committing to an implementation.
- **Trackable execution**: the plan's checkboxes make progress visible and let you resume (or delegate to an agent) without losing the thread.
- **Historical record**: date + topic leave a chronological trail of decisions.
- **It fits the backlog**: idea → spec → plan → ship is the full lifecycle of every item.
