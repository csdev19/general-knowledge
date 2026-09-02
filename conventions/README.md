# Conventions

Reusable, product-agnostic conventions and workflow patterns distilled from real project docs.

## Writing

- **[english-only.md](./english-only.md)** — Everything written into a repo is English: docs, code, comments, commits, PRs, and the tokens inside status legends and tables. The only exception is localized product copy. Includes the checklist for converting an existing repo.

## Architecture patterns

- **[constants-pattern.md](./constants-pattern.md)** — Type-safe enum-like constants: const object + derived type + values array + type guard.
- **[enums-as-const.md](./enums-as-const.md)** — Never use TS `enum`; use `as const` arrays/objects with types derived via `typeof`.
- **[schemas-first.md](./schemas-first.md)** — Define Zod (v4) schemas in the domain package and reuse them across backend and frontend.
- **[tagging-system.md](./tagging-system.md)** — Two-tier tags: default tags in code (slug/label/emoji/color) + per-scope custom tags with a curated color palette. One source of truth across web/mobile/widget.

## Data & delivery

- **[data-sourcing-and-seeding.md](./data-sourcing-and-seeding.md)** — Filling an app with real, structured, geo-tagged reference data (e.g. attractions): why an agent beats a plain chat, legal sources (Wikidata CC0 primary, never scrape TripAdvisor), the curate→resolve→enrich→normalize→validate→seed pipeline, and provenance as a field.
- **[ci-cd-pipeline-strategy.md](./ci-cd-pipeline-strategy.md)** — Trunk + tags, one `verify` gate, tiered checks; the native/mobile install is the biggest CI cost (split it out); `deps:weight` protocol to watch dependency weight.

## Workflow conventions

- **[mvp-first-then-refactor.md](./mvp-first-then-refactor.md)** — Two-phase feature workflow: ship an inline MVP end-to-end first, then extract the domain/application/infra layers once it's stable. When to refactor.
- **[backlog-pattern.md](./backlog-pattern.md)** — How a `backlog/` folder maps deferred work: status legend, index table, one item per file.
- **[changelog-pattern.md](./changelog-pattern.md)** — Docs-app changelog as a decision journal: one file per entry, auto-generated index (no manual index = no merge conflicts), entry bar, and how it complements release-please.
- **[specs-and-plans-workflow.md](./specs-and-plans-workflow.md)** — The brainstorm → spec → plan → archive flow, naming convention, and how it pairs with the backlog.
- **[plan-to-backlog.md](./plan-to-backlog.md)** — Converting an approved plan into self-sufficient backlog deliverables that runner agents execute in parallel; the plan becomes superseded.

## Agent delegation

- **[ai-agent-delegation.md](./ai-agent-delegation.md)** — Two delegation modes (fast/expensive vs slow/cheap), which roles never get cheaper, how to dispatch without blowing up the orchestrator's context, and measured evidence that process structure — not model tier — is what makes delegated work slow. Includes the "name the unit before quoting the number" measurement lesson.
