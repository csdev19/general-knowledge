# Writing Effect: patterns and traps

_The things that are not obvious from the docs, and cost a review round to find._

For how services and layers wire an application layer with repositories and two
consumers, see
[ADR-0001](../architecture/decisions/adr-0001-effect-use-cases.md). This doc is
the complementary half: the patterns for wrapping heavy external dependencies,
and the traps that produce code which looks right and is not.

## The lazy-service pattern

When a service wraps something expensive to load — a compiler, a heavyweight
SDK, anything behind a dynamic `import()` — make the **service value an effect
that yields the dependency**, not the dependency itself.

```ts
// The tag holds a loader, not the module.
export class Compiler extends Context.Tag('app/Compiler')<
  Compiler,
  Effect.Effect<CompilerModule, DependencyLoadError>
>() {}

export const CompilerLive = Layer.effect(
  Compiler,
  Effect.cached(
    Effect.tryPromise({
      try: () => import('some-heavy-module'),
      catch: (cause) => new DependencyLoadError({ dependency: 'some-heavy-module', cause }),
    }),
  ),
)
```

That one indirection does three jobs at once:

1. **The load stays lazy.** Building the layer only prepares the loader. Nothing
   is imported until a program actually asks for the dependency.
2. **The layer cannot fail on construction.** Its own error channel is empty, so
   a runtime built from it cannot fail while being built. Failures surface where
   the dependency is used, which is where you can do something about them.
3. **The domain error channel stays narrow.** The program maps
   `DependencyLoadError` to *its* error at the point of use:

```ts
const compile = (src: string) =>
  Effect.gen(function* () {
    const load = yield* Compiler
    const compiler = yield* Effect.mapError(
      load,
      (cause) => new ParseError({ message: `could not load the compiler: ${cause.message}` }),
    )
    // …
  })
// Type stays: Effect<Result, ParseError, Compiler>
```

The payoff is that the published signature never widens. A whole package can
move to services without a single observable error contract changing.

The naive alternative — `Layer.effect(Compiler, Effect.tryPromise(...))` with
the module as the service value — loads eagerly when the layer is built and puts
the load error into the layer's channel, where every consumer has to deal with
it.

## A declared error type is a lie until you close both holes

`Effect<A, MyError>` claims `MyError` is the only way it fails. Two things break
that claim silently, and both look fine in review.

**Hole 1: `Effect.promise`.** It assumes the promise never rejects. A rejection
becomes a *defect* — invisible to `catchTag`, surfacing to the caller as a raw
`FiberFailure`. Use `Effect.tryPromise` with an explicit `catch`. Treat
`Effect.promise` as an assertion that rejection is impossible, and write it only
when that is actually true.

**Hole 2: synchronous throws inside `Effect.gen`.** Anything between the
`yield*`s is ordinary JavaScript. A throw from a third-party call, a stack
overflow in a recursive walk, a `TypeError` — all become defects.

```ts
const program = Effect.gen(function* () {
  const compiler = yield* Compiler
  return walkTheAst(compiler)   // plain JS. A throw here is a defect.
}).pipe(
  // For an entry point whose whole job is "this can fail to parse", converting
  // defects is honest. Do NOT do this blanket-style over business logic —
  // there you are hiding bugs.
  Effect.catchAllDefect((defect) =>
    Effect.fail(new ParseError({ message: reasonOf(defect) })),
  ),
)
```

The test for whether your error channel is honest is not "does it compile" — it
is "can I write a test that provides a failing dependency and catch it with
`catchTag`?" If the test sees a `FiberFailure`, the type is lying.

## `Effect.cached` caches failures too

`Effect.cached` memoises the *outcome*, success or failure, for the life of the
layer. That is right for an immutable installed module: it will not start
existing later.

It is wrong for anything that can recover while the process lives — a network
fetch, a connection, a token. There, a transient failure is cached and every
later call in that runtime gets the stale failure.

If a dependency can recover, make retry an explicit policy
(`Effect.retry(Schedule…)`), not an accident.

## Runtime ownership

Someone must own the runtime. There are two defensible answers and one common
mistake.

**Module-local runtime**, created lazily and never disposed. Acceptable **only**
when every layer in it holds lazy imports or pure values — nothing with a
finalizer worth running.

```ts
let instance: Runtime | undefined
export const runtime = () => (instance ??= ManagedRuntime.make(AppLayers))
```

The moment a layer acquires a real resource — a pool, a worker, a telemetry
exporter — this is wrong, because nothing ever disposes it. Write that condition
into the code as a comment next to the runtime, not only into a doc. It will be
violated silently otherwise.

**Process-owned runtime.** The entry point (server, CLI, worker) builds the
runtime at startup and disposes it on shutdown. This is what you need for real
resources, and it is also what level 2 of
[adoption-strategy.md](./adoption-strategy.md) requires.

**The mistake: building a runtime per call.** It rebuilds every layer each time,
so every memoisation is defeated and every acquire/release cycles. If you find
`Effect.provide(SomeLive)` inside a hot path, that is this bug.

Per-request layer construction is a middle case and can be fine — see the
neutral note in
[ADR-0001](../architecture/decisions/adr-0001-effect-use-cases.md), where
per-request repository construction is acceptable precisely because those
constructors do no I/O. The rule generalises: **per-request construction is fine
when construction is free.** Verify that rather than assuming it.

## One merged runtime couples the laziness of every layer

A runtime built from `Layer.mergeAll(A, B, C)` builds all three when it first
runs anything. If any one of them loads eagerly at build time, then using *only*
A also loads B and C.

This is the failure mode that appears the moment you have more than one lazy
service, and it does not exist while you have one. If lazy loading matters —
startup time, an optional heavy dependency — assert it **per entry point**, not
once:

```
parsing        → loads only the compiler
generating     → loads only the data library
printing       → loads only the printer
```

## Test with layers, not with module mocks

The concrete payoff of services is that test doubles stop being module-loader
tricks:

```ts
// Before: mocking the module system.
vi.doMock('typescript', () => { throw new Error('boom') })

// After: providing a different implementation.
const Broken = Layer.succeed(Compiler, Effect.fail(new DependencyLoadError(…)))
const realWithOneThrow = Layer.effect(Compiler, Effect.cached(
  Effect.promise(async () => ({ ...(await import('typescript')).default,
    createProgram: () => { throw new Error('exploded') } })),
))
```

The second form tests your seams instead of the test framework's interception,
and it makes failure cases cheap enough that you actually cover them.

### The trap that hides inside this

If you keep a hoisted module mock anywhere (e.g. to assert *whether* a module was
loaded), know that **a hoisted `vi.mock` factory runs once per test file**, and
`vi.resetModules()` does not re-run it. Any recorder inside that factory records
only the first evaluation. Every later assertion in the file reads a stale value
and silently asserts nothing.

The fix is a per-test `vi.doMock` after `vi.resetModules()`. The general lesson
is bigger than Vitest:

> A guard that has never been red is not a guard. After writing a test that
> protects an invariant, break the invariant on purpose, watch the test fail,
> and revert. This is the only way to catch a test that passes for the wrong
> reason — and a test that passes for the wrong reason is worse than no test,
> because it buys confidence it has not earned.

## `Scope` is for real resources, not ceremony

`acquireRelease` is worth it when release does something. Before wrapping a
third-party object in a scope, check that it actually exposes a
`close`/`dispose`/`release`. Plenty do not — they are simply garbage-collected —
and wrapping them produces an `acquireRelease` with an empty release: pure
ceremony that reads as resource safety in review.

One `grep` of the type definitions answers this. Do it before writing the scope,
not after.

## Do not over-claim caching

"The layer memoises it" is usually a smaller claim than it sounds. For a dynamic
`import()`, the module registry already caches the module; the layer makes it
explicit and shares one reference. That is worth having, but it is not new
caching.

Objects derived from *inputs* — a compiler program built from one file's source,
a client configured for one request — are not shareable at all and must stay
per-call. Be precise about which of the two you have, in the code comment as
well as in your head.

## Checklist before opening the PR

- [ ] Every `Effect.promise` is a deliberate "this cannot reject" assertion.
- [ ] Synchronous work inside `Effect.gen` cannot escape as a defect past an
      entry point that declares a typed error.
- [ ] Layer error channels are empty, or the runtime's failure mode is handled.
- [ ] `Effect.cached` is only on things that cannot recover.
- [ ] The runtime owner is explicit, and the condition making it valid is written
      next to it.
- [ ] Laziness, if it matters, is asserted per entry point.
- [ ] Every invariant test has been observed failing.
- [ ] Exported programs with a non-empty `R` also export their tags and layers.

## Related

- [why-effect.md](./why-effect.md) — whether to adopt at all.
- [adoption-strategy.md](./adoption-strategy.md) — how far, and in what order.
- [ADR-0001: Effect for application use cases](../architecture/decisions/adr-0001-effect-use-cases.md)
  — services and layers for an application layer with two consumers.
- [error-handling/result-types.md](../error-handling/result-types.md) — the
  lighter alternative when you only want typed errors.
