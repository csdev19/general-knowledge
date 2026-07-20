# Conventions

Reusable, product-agnostic conventions and workflow patterns distilled from real project docs.

## Architecture patterns

- **[constants-pattern.md](./constants-pattern.md)** — Type-safe enum-like constants: const object + derived type + values array + type guard.
- **[enums-as-const.md](./enums-as-const.md)** — Never use TS `enum`; use `as const` arrays/objects with types derived via `typeof`.
- **[schemas-first.md](./schemas-first.md)** — Define Zod (v4) schemas in the domain package and reuse them across backend and frontend.
- **[tagging-system.md](./tagging-system.md)** — Two-tier tags: default tags in code (slug/label/emoji/color) + per-scope custom tags with a curated color palette. One source of truth across web/mobile/widget.

## Workflow conventions

- **[mvp-first-then-refactor.md](./mvp-first-then-refactor.md)** — Two-phase feature workflow: ship an inline MVP end-to-end first, then extract the domain/application/infra layers once it's stable. When to refactor.
- **[backlog-pattern.md](./backlog-pattern.md)** — How a `backlog/` folder maps deferred work: status legend, index table, one item per file. _(Spanish)_
- **[specs-and-plans-workflow.md](./specs-and-plans-workflow.md)** — The brainstorm → spec → plan → archive flow, naming convention, and how it pairs with the backlog. _(Spanish)_
