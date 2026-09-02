# Backlog Pattern

_How to keep a `backlog/` folder that is the source of truth for "what's coming", without ever losing deferred work._

A `backlog/` is a documentation folder (not a ticket board) where all **planned, proposed or paused work that is not finished yet** lives. It is the map of "what's coming", with enough context on each item to pick it back up without re-investigating.

The core idea: when you postpone an idea, an improvement or an analysis, **it is not lost**. It stays written down with its context. When you come back to it weeks later, you do not start from zero.

## Anatomy

```
backlog/
├── index.md              # The map: a table with every item and its status
├── <feature-a>.md        # One item = one file, with the context to resume it
├── <feature-b>.md
└── ...
```

- **`index.md`** — the map table. The source of truth for "what's next".
- **One item = one file** — every item with substance earns its own doc. Trivial items (just an idea, ⚪) can live only as a row on the map, with no doc of their own.

## Status legend

Emojis so you can scan status at a glance:

| Emoji | Meaning                                  |
| ----- | ---------------------------------------- |
| 🟡    | In progress (possibly paused)            |
| 🟢    | Ready to validate (merged / in prod)     |
| 🔵    | Proposed (analyzed, not started yet)     |
| ⚪    | Idea (no analysis yet)                   |

It can be extended with terminal statuses (`✅ Decided and closed`) or watch statuses (`Keep an eye on`), but these four are the base. An item that is `✅` and already validated is **removed from the map**, and its learning moves into the matching feature/architecture doc.

## Map format (the index table)

Each row is an item. Minimum columns: **Item · Area · Status · Effort · Detail (link)**.

```markdown
| Item                            | Area      | Status                       | Effort     | Detail             |
| ------------------------------- | --------- | ---------------------------- | ---------- | ------------------ |
| **Item name** (context)         | Pipeline  | 🟢 Merged (#12) · validate   | Medium     | [View](./my-item)  |
| Another, smaller item           | Editor    | 🔵 Proposed                  | Low–Medium | [View](./other)    |
| A loose idea                    | Infra     | ⚪ Idea                      | High       | —                  |
```

- **Area** groups by domain/subsystem (Pipeline, Editor, Infra, QA, DX...).
- **Status** is the emoji + a short note (PR number, "validate in prod", etc.).
- **Effort** is a coarse estimate (`Low` / `Medium` / `High` / `Mixed` / `—`).
- **Detail** links to the item's file, or `—` if it has no doc of its own.

## Item format (one file)

Each item file needs **just enough to resume it without re-investigating**: what it is, what state it was left in, what was decided, what is missing.

```markdown
# <Item name>

> **Status: <emoji> <one-line summary>.** Where it was left, what is missing to close it.
> Links to the spec/plan if they exist.

One or two paragraphs: what problem it solves and why it matters.

## What was done / what shipped

| Piece        | What it does                      |
| ------------ | --------------------------------- |
| ...          | ...                               |

## What is missing / how to validate

- [ ] Pending step 1
- [ ] Pending step 2

## Out of scope (separate items)

- Related thing → link to its own backlog item.
```

What matters is not the exact template but the principle: **the file has to leave the next reader (or you, a month from now) ready to act**, with no re-investigation.

## How it is used

- Every item that gets picked up moves to 🟡 and, if it has substance, earns its own doc.
- When an item is completed: move the learning into the matching feature/architecture doc and **remove it from the map**.
- Keep **effort** and **status** always up to date — it is the source of truth for "what's next".

## Why it helps

- **Deferred work is never lost.** Every postponed idea or improvement stays written down with its context.
- **Resuming is free.** The context lives in the item, not in the head of whoever paused it.
- **Status is visible at a glance.** The map with its emojis says what is next without opening anything.
- **It separates "thinking" from "doing".** A 🔵 proposed item already has the analysis; starting it is just execution.
