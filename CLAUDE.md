# Development Rules

This repository is the **general-knowledge hub**: the single source of truth for reusable,
product-agnostic knowledge (architecture, stacks, conventions, error handling, monorepos,
packages). Templates and projects **link here** instead of copying docs around.

## English is the only language of this repository

**Everything written here is in English — no exceptions.** Docs, headings, tables, status
legends, code samples, comments, commit messages, branch names, PR titles and PR bodies.

This holds even when the conversation with the user happens in Spanish: reply in whatever
language they use, but **write English into the repo**. A Spanish source (a vault note, a chat
thread) gets translated on the way in, never pasted.

The full rule, the rationale, and the checklist for applying it to an existing repo:
[conventions/english-only.md](./conventions/english-only.md). Every project that links to this
hub inherits it.

## How this repo is organized

Two layers, both documented in [README.md](./README.md):

- **Flat knowledge by topic** — the source of truth, reusable and mostly stack-agnostic. Each
  folder has its own `README.md` index.
- **[`stacks/`](./stacks/)** — recipes that **compose** the flat topics into a "how to build with
  this stack" guide. They never duplicate content: they link, and add the assembly notes specific
  to that stack.

## Writing rules

- **Never duplicate content across docs.** Link to the doc that owns the topic. If two docs need
  the same explanation, one of them is the owner and the other links to it.
- **Every new doc gets an index row** in its folder's `README.md`, and — when it is worth adopting
  from day one — a line in the root `README.md`.
- **Keep docs product-agnostic.** Real project names appear only as examples, clearly marked as
  such; the pattern itself must be reusable.
- **Docs record decisions, not just APIs.** Say what was chosen, what was rejected, and why —
  that is the part a reader cannot reconstruct from the code.
