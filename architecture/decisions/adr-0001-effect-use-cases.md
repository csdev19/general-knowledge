# ADR 0001: Effect for Application Use Cases

_Adopt Effect-TS as the orchestration layer for all application use cases, replacing async `Result<T>`._

## Status

| Field         | Value      |
| ------------- | ---------- |
| Status        | Accepted   |
| Supersedes    | N/A        |
| Superseded by | N/A        |

## Context

The application layer originally used plain async functions returning `Result<T, E>` — a discriminated union `{ data: T; error: null } | { data: null; error: E }`. This worked for simple use cases but had three growing problems:

1. **Dependency injection was ad-hoc.** Each use case accepted a `repos` object as a positional parameter. As the repository count grew (to ~10 across several modules), call sites became verbose and changing the parameter shape broke every consumer.

2. **Error propagation was manual.** Nested use case calls required threading `Result` through each layer by hand, or abandoning type safety with try/catch.

3. **Dual consumers needed identical wiring.** Both the web app's server functions and the API's routes call the same use cases but map errors to different formats (a `withErrorHandling` envelope vs. a typed RPC error). The `Result` pattern had no standard bridge; each consumer re-implemented the mapping ad hoc.

## Decision

Adopt **Effect-TS** for all application use cases:

- `Effect.gen` for the use case body.
- `Context.Service` for dependency injection (repository tags in `packages/application/src/services.ts`).
- `Data.TaggedError` for all domain errors.
- `Layer.succeed` + `Layer.mergeAll` via `createAppLayer(db)` for consumer-level wiring.
- A `runUseCase` helper per consumer to bridge Effect to that consumer's error format.

## Consequences

**Positive:**

- Dependency injection is structural — use cases declare requirements as `Context.Service` tags; `createAppLayer` satisfies all of them at once; adding a new repo to a use case requires no call-site changes.
- Typed error channel — the compiler enforces that all tagged errors are handled in `run-use-case.ts`; new errors added to a use case produce a type error at the bridge until the switch is updated.
- Composable — use cases can call other use cases with `yield*` inside `Effect.gen` without losing type information.

**Negative:**

- `effect` is a large dependency; it is server-only but increases the application package footprint.
- A beta API can be unstable — some helpers were removed between beta versions, requiring test rewrites when upgrading.
- Learning curve for contributors unfamiliar with functional effect systems.

**Neutral:**

- `createAppLayer(db)` instantiates all repo classes on every request, even when a use case only needs a couple. Repository constructors are synchronous and trivial — no I/O at construction time — so this is not a performance concern.

## Alternatives Considered

### Keep async `Result<T, E>`

The existing pattern. Works for simple cases, but the `repos` parameter grows with the repo count and there is no structural guarantee that all error cases are handled. Rejected because the dual-consumer requirement made the wiring duplication worse over time.

### Neverthrow / true-myth

Typed `Result` libraries without a full effect system. Solve the error-channel problem but not the dependency injection problem — repos still need to be passed as explicit parameters. Rejected.

### Inversify / tsyringe (IoC containers)

Traditional runtime DI containers for TypeScript. Solve DI but require container setup in tests and introduce a separate paradigm without a typed error channel. Rejected in favor of Effect's built-in, compile-time `Context` system.

## References

- [Domain Modeling Strategy](../domain-modeling-strategy.md) — How value objects and use cases enforce invariants with Effect.
- [Application Layer](../application-layer.md) — Overall layer architecture and dependency rules.
