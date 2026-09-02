# Effect Knowledge Hub

Effect as a default tool for TypeScript backends: whether to adopt it, how far
to let it reach, and how to write it without the traps.

> Scope: the adoption decision and the patterns that survive contact with a real
> codebase. For the worked example of wiring an application layer — repositories
> as services, two consumers, a `runUseCase` bridge — see
> [ADR-0001](../architecture/decisions/adr-0001-effect-use-cases.md), which this
> topic complements rather than repeats.

## Contents

| File | Summary |
| --- | --- |
| [why-effect.md](./why-effect.md) | What `Effect<A, E, R>` actually is, the six things it buys ranked by real payoff, what it explicitly does not buy, the costs including the **half-adopted trap**, how it compares to a `Result` type, and when adoption is and is not worth it. |
| [adoption-strategy.md](./adoption-strategy.md) | Adoption as a decision about **reach**, not volume: why what propagates is runtime ownership rather than the import, the three nested levels, the single capability (interruption) that forces the boundary to move, how to choose the first package, an increment order that stays green, and the public-API rule for exported `R`. |
| [writing-effect.md](./writing-effect.md) | The patterns and traps: the **lazy-service pattern**, the two holes that make a declared error type a lie, `Effect.cached` caching failures, runtime ownership, how one merged runtime couples every layer's laziness, testing with layers instead of module mocks, and why `Scope` is often ceremony. Ends with a pre-PR checklist. |

## The short version

- Effect's value is **errors and dependencies in the type**, plus interruption
  and resource safety. Typed errors alone are cheaper with a
  [`Result`](../error-handling/result-types.md).
- The worst place to be is **half-adopted** — Effect as a `Promise` wrapper with
  tagged errors pays the full conceptual cost for almost none of the capability.
- Adoption is about **who owns the runtime**. Start inside one leaf package with
  Promise facades intact; only cancellation forces the boundary outward.
- **Do not migrate synchronous packages.** Effect over sync code buys the
  service graph and none of the async machinery that justifies its weight.
- **Prove every invariant test can fail.** A guard that has never been red is
  not a guard.

## Related

- [architecture/decisions/adr-0001-effect-use-cases.md](../architecture/decisions/adr-0001-effect-use-cases.md)
  — adopting Effect for application use cases: `Context.Service` tags for
  repositories, `createAppLayer`, and a per-consumer error bridge.
- [architecture/observability.md](../architecture/observability.md) — where
  `Effect.withSpan` and OpenTelemetry fit in the long-term observability plan.
- [architecture/domain-modeling-strategy.md](../architecture/domain-modeling-strategy.md)
  — Effect use cases enforcing domain invariants.
- [error-handling/](../error-handling/) — the `Result` pattern and the
  centralized error-handling wrapper Effect would eventually replace.
