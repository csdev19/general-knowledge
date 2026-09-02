# Why Effect

_What Effect actually is, what it buys, what it costs, and when it is worth it._

## What it is

Effect is one type and a runtime that interprets it.

```ts
Effect<A, E, R>
//     │  │  └─ Requirements: what services this program needs to run
//     │  └──── Error: how it can fail, in the type
//     └─────── Value: what it produces on success
```

An `Effect` is a **description** of work, not the work itself. Nothing runs
until a runtime executes it. That indirection is the whole trick: because the
program is a value, the runtime can retry it, time it out, interrupt it, trace
it, and supply its dependencies — none of which is possible with a `Promise`,
which is already running by the time you hold it.

Compare the two signatures for the same function:

```ts
// What TypeScript normally lets you say
declare function parseModel(source: string): Promise<Model>

// What Effect lets you say
declare function parseModel(source: string): Effect<Model, ParseError, Compiler>
```

The second one says: this can fail, and only in this way; and it cannot run
unless someone provides a compiler. Both facts are now checked by the compiler
instead of living in a docstring.

## What it buys, in order of real payoff

**1. Errors in the type.** `Promise<T>` says nothing about failure. Every
`catch` is `unknown`, and every failure mode is documentation at best. Effect's
`E` channel is exhaustive and `catchTag` narrows it. This is the benefit you
feel on day one.

**2. Dependencies in the type.** `R` is dependency injection with no container
and no decorators: a program declares what it needs, and it will not type-check
until someone supplies it. In tests, "supply it" means passing a different
layer — no module-loader mocking, no `jest.mock`, no import interception.

**3. Interruption.** Structured concurrency: when a fiber is interrupted,
everything it started is interrupted too, and finalizers run. This is the one
capability with no reasonable approximation in Promise-land — a `Promise` cannot
be cancelled, only ignored.

**4. Resource safety.** `Scope` and `acquireRelease` guarantee a release runs
even on interruption. Meaningful only when you have a real resource with a real
release; see the trap in [writing-effect.md](./writing-effect.md).

**5. Observability.** Spans and structured logging are part of the runtime
rather than a wrapper you remember to apply.

**6. Policy as data.** `timeout`, `retry` with a `Schedule`, `race`, rate limits
— composable and testable, instead of hand-rolled per call site.

## What it does not buy

Be honest about this or the adoption argument collapses on first contact.

- **It does not improve synchronous code much.** Over a pure sync module you get
  the error channel and the service graph, and none of the async machinery that
  justifies most of Effect's weight.
- **It is not validation.** Runtime schema work is a separate library
  (`effect/Schema`) and a separate decision. Adopting Effect does not mean
  replacing Zod.
- **It does not reduce bundle size.** It adds to it.
- **It does not make bad boundaries good.** A tangled module is still tangled
  once it returns `Effect`.

## What it costs

**Conceptual surface.** `Context`, `Layer`, `Scope`, `Runtime`, `Fiber`,
`Schedule` — a contributor who only wants to change a handler now has to read
several of them. The cost is not learning Effect once; it is that every future
contributor to that code pays the same toll.

**Interop friction.** Every Promise-based library needs wrapping at the edge.
That is a small amount of code each time and a constant tax on velocity.

**Debugging through a runtime.** Stack traces run through the fiber runtime.
`Cause` is more informative than a stack once you learn to read it, and worse
than a stack until then.

**The half-adopted trap — the expensive one.** Importing Effect and using it as
a `Promise` wrapper with tagged errors is the worst position available: you pay
the full conceptual surface and collect almost none of the capabilities. It is
also the position a codebase drifts into by default, because `tryPromise` +
`runPromise` is the smallest possible change and it works.

If a codebase is here, the fix is to pick a direction, not to add more Effect
uniformly. [adoption-strategy.md](./adoption-strategy.md) is about picking it.

## Effect versus a `Result` type

This hub already documents [`Result<T, E>`](../error-handling/result-types.md),
and the overlap is real: both put errors in the type instead of in `throw`.

| | `Result<T, E>` | Effect |
| --- | --- | --- |
| Typed errors | Yes | Yes |
| Runtime cost | None — a discriminated union | A fiber runtime |
| Dependency injection | No | Yes, via `R` |
| Interruption | No | Yes |
| Composition | Manual (`map`/`flatMap` helpers you own) | Built in |
| Onboarding | Minutes | Weeks |

`Result` is the 20% that delivers typed errors for almost nothing. Reach for
Effect when you want the other capabilities, not because typed errors sound
appealing — `Result` already gives you those.

A codebase can hold both: `Result` in synchronous domain code, Effect where
concurrency, resources and injection actually live. What it should not do is
convert between them at every call site.

## When it is worth it

Worth it when several of these are true:

- Real concurrency: parallel work, races, work that should stop when the caller
  goes away.
- Real resources: connections, file handles, subprocesses, anything with a
  release that must run on the failure path too.
- Dependencies that are painful to fake: a compiler, a clock, an SDK client, a
  heavyweight module you would otherwise mock through the module loader.
- Failure modes worth enumerating: a surface where "what can go wrong here" is a
  question people actually ask.
- Long-lived code with several contributors, where the type-level contract earns
  its onboarding cost back.

Not worth it when:

- The module is synchronous and stays that way.
- The error channel is already small, explicit and readable.
- It is a script, a one-off, or code with a known short life.
- Nobody on the team has used Effect and the code is not important enough to
  justify the first person learning it here.

## The ecosystem argument

Effect has moved from "a functional-programming curiosity" to something a
TypeScript backend engineer is expected to at least read. That matters for two
practical reasons: hiring and library choice both get easier the closer you sit
to where the ecosystem is heading, and knowledge acquired here transfers out.

It is a real argument, and it is not a sufficient one on its own. Adopt it where
the capabilities above pay for themselves, and the familiarity comes free with
that. Adopting it because it is popular produces exactly the half-adopted state
described above — the most expensive place to be.

## Next

- [adoption-strategy.md](./adoption-strategy.md) — how far it should reach, and
  how to get there incrementally.
- [writing-effect.md](./writing-effect.md) — the patterns and the traps.
