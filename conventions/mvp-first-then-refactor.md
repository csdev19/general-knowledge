# MVP first, then refactor to the architecture

_A two-phase workflow for adding features to a DDD + hexagonal codebase: make it work end-to-end the
simplest way possible, then extract the layers once it's stable._

The layered architecture (domain ← application ← infra, apps wire them) is the destination, not the
starting point. Reaching for use cases, domain interfaces, and mappers before the feature even works
is premature — you design abstractions against requirements you don't understand yet. This workflow
front-loads a working slice, then pays down the structure.

## Phase 1 — MVP (speed)

Build the feature the simplest way that works. All logic can live inline in the frontend and backend:

- Put business logic directly in the consumer — an API route handler, or a server function / data
  hook in the app.
- Use the infrastructure repositories (e.g. the DB package) directly from the handler.
- Focus on making it work end-to-end: **UI → API → DB**.
- No use cases, domain interfaces, or mappers yet.

The goal of this phase is a working vertical slice you can see and test, not clean layering.

## Phase 2 — Refactor to the architecture

Once the feature works, extract it into the layers, in dependency order:

1. **Domain** (`packages/domain/`) — interfaces, schemas, types, constants.
2. **Application** (`packages/application/`) — use cases that depend only on domain interfaces.
3. **Infrastructure** (`packages/infra-*`) — repository implementations, mappers.
4. **Consumers** (`apps/*`) — thin handlers that wire application use cases to infra
   implementations.

The dependency rule stays strict: **domain ← application ← infra**, and only apps wire them
together.

## When to refactor

Pull the trigger on Phase 2 when any of these becomes true:

- A **second consumer** needs the same logic (e.g. both a web server function and an API route).
- Business logic in a handler exceeds **~15 lines**.
- The feature is **stable and tested** — you now understand the real shape of the abstraction.

Until one of those holds, the inline MVP is the correct amount of structure.
