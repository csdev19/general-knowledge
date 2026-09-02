# Adopting Effect incrementally

_Adoption is a decision about reach, not about how much Effect you import._

## The rule that governs everything

When a package goes Effect-first, what spreads outward is **not** the
`import { Effect } from 'effect'`. That part is trivial. What spreads is the
third type parameter.

An `Effect<A, E, R>` declares what it requires. The moment a program asks for a
service, its type says so, and nobody can run it without providing that service.
The obligation climbs the call stack until it reaches whoever calls
`runPromise` — and **that** party has to own a runtime built from all the layers.

So the real question is never "how much Effect?" It is: **who owns the runtime,
and where does the boundary sit?**

## The three levels of reach

These nest. Each contains the one before it.

### Level 1 — Effect-first inside one library

The package uses services, layers and typed error channels internally, and keeps
exporting Promise-shaped functions. Consumers are unchanged and do not know
Effect is involved.

- **Buys:** dependency injection, typed errors, internal timeouts, resource
  scopes, spans wired but inert.
- **Does not buy:** interruption from the caller, end-to-end traces.
- **Costs:** only contributors to that package pay the learning cost.

### Level 2 — Effect crosses into the adapters

Effects become the primary API. The HTTP layer, the CLI, the job worker — each
builds a runtime once at startup, runs programs through it, and translates typed
failures at its own border.

- **Buys:** real interruption when a request aborts or a connection closes; one
  trace per request.
- **Costs:** every contributor to those adapters now needs Effect to change a
  handler. This is the real price, and it is easy to under-count because it is
  paid later and by other people.

### Level 3 — one error model for the whole product

Domain and persistence packages convert too. Hand-rolled `Result`-style channels
become Effect's error channel.

- **Buys:** coherence — one way to fail, everywhere.
- **Costs:** the largest rewrite, over the packages with the deepest test
  surface, and usually the least capability gained per line changed.

## Which capability forces the boundary to move

This is the question that actually decides the level, and it has a sharp answer.

| Capability | Reachable at level 1? |
| --- | --- |
| Services and layers (DI) | Yes |
| Resource scopes | Yes |
| Clock injection | Yes |
| Timeouts | Yes |
| Trace spans | Yes, but only useful once something consumes them |
| **Interruption** | **No** |

A Promise handed back by `runPromise` gives the caller nothing to interrupt. The
fiber underneath keeps running whether or not anybody is still waiting. If you
want in-flight work to stop when the client disconnects, the caller must hold
the fiber — and that means level 2.

So: **default to level 1, and move to level 2 only when you can name the work
that should have been cancelled and what it cost you.** "It would be cleaner" is
not that.

## Choosing the first package

Pick a leaf. Specifically:

- **No internal dependencies.** A package that other packages depend on, but
  which depends on nothing internal itself, can be migrated without touching
  anything else. This is the single biggest predictor of whether the migration
  stays contained.
- **Real external dependencies worth injecting.** A heavyweight SDK, a compiler,
  an HTTP client — something whose test double is currently painful. If there is
  nothing to inject, level 1 buys you very little.
- **A meaningful failure surface.** If everything either works or throws once,
  the error channel is not earning anything.
- **Its own test suite.** You want the safety net before you start.

Avoid starting in the package everyone edits daily. The first migration is where
you learn the patterns and get them wrong once.

## An increment order that stays green

Each step is independently committable and leaves the suite passing.

1. **Write the invariant tests first** — before extracting anything. Whatever
   property the migration could silently break (lazy loading, startup cost, an
   error contract) gets a test now, while the old behaviour is still there to
   characterise. Then **prove the test can fail** by breaking the property on
   purpose and reverting. A guard that has never been red is not a guard.
2. **Extract one service** and move its consumer onto it. Combine the service
   and its first consumer in one step — a layer with no consumer is dead code.
3. **Repeat per dependency.** The second and third are mechanical once the first
   is right.
4. **Add spans and timeouts only where a measurement justifies them.** Spans
   nobody reads are decoration.

Do not do them all at once. The type errors from a half-migrated `R` are the
signal that tells you which call sites you forgot; that signal is only useful in
small batches.

## The public-API rule

If a package exports a program whose type includes `R`, it **must** also export
the tags and live layers that satisfy it.

```ts
// Exported, but the consumer has no way to provide TypeScriptCompiler.
// The published type is unsatisfiable — this is a broken public surface.
export const parseTypesEffect: Effect<Model, ParseError, TypeScriptCompiler>
```

Either export the tags and layers beside it, or do not export the program at
all. There is no coherent third option.

What should *not* be exported is the package's own internal runtime. A consumer
building Effect programs should own its runtime; handing them yours couples
their lifecycle to your module scope.

## What not to migrate

**Fully synchronous packages.** If a package contains no `async`, no `await` and
no `Promise`, Effect buys the error channel and the service graph but not the
async machinery that justifies most of its weight. Check this before assuming —
it is easy to be wrong about, and it is one `grep` away.

**Small, readable error channels.** A three-line discriminated union that every
contributor can read is not obviously worse than a typed Effect channel. It is
worse only if you need the composition.

**Anything whose migration you cannot describe as a capability.** If the answer
to "what does this let us do that we could not do before?" is "it is more
consistent", the work is coherence, not capability. Schedule it accordingly —
which usually means not now.

## Recording the decision

Adoption is an architecture decision and should leave a record: what was chosen,
which levels were rejected, and the measurement that killed each.
[ADR-0001](../architecture/decisions/adr-0001-effect-use-cases.md) is the worked
example in this hub.

Two things worth writing down explicitly, because they are the ones people
re-litigate later:

- The **level** chosen, and the specific capability that would justify moving to
  the next one.
- The **constraint on the runtime owner** — a module-local runtime is only safe
  while its layers hold nothing that needs disposing. That is a rule that will be
  broken silently unless it lives in the code as well as the doc.

## Next

- [writing-effect.md](./writing-effect.md) — the patterns and the traps you hit
  while doing this.
