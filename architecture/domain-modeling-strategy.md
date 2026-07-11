# Domain Modeling Strategy

_A pragmatic middle-path DDD: contract types as wire truth, branded IDs in the domain, value objects only for high-leverage primitives, entities documented as markdown, and Effect use cases enforcing invariants._

## TL;DR

A **middle-path DDD** designed to keep cognitive overhead low while preserving the parts that pay off:

1. The **API contract package** is the source of truth for types on the wire. Web and server consume the same contract types.
2. **Branded ID types** in `@app/domain` give compile-time safety with zero runtime cost.
3. **Value Objects** are reserved for primitives whose invariants go beyond "it is a string" — `Money`, `InviteCode`, `Email`. Everything else stays as branded primitives.
4. **Entities are documented as markdown** in `packages/domain/docs/entities/`, not implemented as classes. Their invariants are enforced by **use cases** in the application layer.
5. **Mutable entities, immutable value objects.** Errors are typed via `Data.TaggedError`. No hand-rolled `Result` validation, no class hierarchies in the domain.

## The Three Layers of Type Truth

Each layer owns a different slice of "what the type is."

| Layer           | Location                                  | Owns                                                                                       |
| --------------- | ----------------------------------------- | ------------------------------------------------------------------------------------------ |
| **Contract**    | `packages/api-contract/src/*.contract.ts` | Wire shape. Inputs, outputs, response codes. Date fields exposed as `string` to consumers. |
| **Domain**      | `packages/domain/src/`                    | Types, branded IDs, schemas, repository interfaces, value object factories.                |
| **Application** | `packages/application/src/`               | Use cases that enforce invariants and orchestrate repos.                                    |

### Why the contract owns the wire

The server returns `new Date()`. `JSON.stringify` calls `Date.prototype.toJSON` and produces an ISO string; `JSON.parse` keeps it as a string. Many RPC clients do not run output validation by default, so a `z.coerce.date()` on the contract is for IDE typing of server handler return types — not runtime client coercion. A "jsonified" client type flips the surface so consumer types match the wire reality (dates are strings on the wire).

The web app, the server that hosts the API, and any future mobile app all consume the same contract types. The server satisfies the contract by TypeScript inference at the router layer — handlers return values that conform to the output schema, and `tsc` proves the conformance.

### Why domain owns the structural types

Branded IDs and value object factories live in `@app/domain` so they survive past any single transport. Contract schemas exist for the wire; domain types exist for the rules. They overlap (an `Order.id` is the same UUID in both) but they are not the same type.

### Why application owns invariant enforcement

Entities have rules — "a member cannot be removed while they have a non-zero balance," "an invoice cannot be marked paid for more than its remaining amount." Those rules are enforced in **use cases** (`packages/application/src/{module}/{use-case}.ts`) rather than entity classes. Use cases use typed errors and dependency injection.

The trade-off is explicit: instead of compile-time invariant enforcement on every entity construction, you get runtime invariant enforcement in use cases plus markdown documentation of every invariant per entity.

## Branded ID Types

All entity IDs in `@app/domain` use branded `string` types. They are compile-time only — `tsc` refuses to mix them, and runtime cost is zero.

### Naming convention

One branded type per entity, named `{Entity}Id`. A representative set:

- `OrderId`
- `UserId`
- `MemberId` (membership row id, distinct from `UserId`)
- `LineItemId`
- `InvoiceId`
- `PaymentId`
- `NotificationId`
- `InviteId`

### When to add a branded ID

Whenever a `string` carries identity. If a function takes `orderId: string` and could be passed `userId: string` by mistake, that is the branded-ID smell. Add the brand instead.

### When NOT to use a branded ID

For things that are merely "a string with no real invariant" — a description field, a free-text note, a slug. Branded IDs encode "this is the right kind of UUID," not "this is a non-empty string." Use schema validation for the second case.

## Value Object Policy

Value objects encapsulate primitives that have **real invariants or behavior** — arithmetic, normalization, formatting, equality. They are immutable.

### When to use a Value Object

- The primitive carries arithmetic that must not be done with the wrong unit (`Money` — never add cents to dollars).
- The primitive has a non-trivial format (`InviteCode` — fixed length, restricted alphabet, a generation algorithm).
- The primitive needs normalization (`Email` — lowercase, trim, mask for logs).
- Equality is structural, not reference-based.

### When NOT to use a Value Object

- The primitive is just a string with a min length. Validate at the boundary, use branded `string` afterward.
- The primitive is a plain `Date` field. Treat it as a `string` on the wire and a `Date` only at the leaf where formatting needs it.
- The primitive is a free-text description. Validate at the boundary, leave it a `string`.

### Recommended initial set

- `Money` — `{ amount: number, currency: Currency }` where `amount` is integer minor units (cents). Methods: `add`, `subtract`, `times`, `eq`, `format`. Refuses cross-currency arithmetic.
- `InviteCode` — wraps a `generateInviteCode()` helper. Validates format on construction; exposes `value` for serialization.
- `Email` — normalized lowercase string. Methods: `mask` (for logs), `domain`, `eq`.

This list grows as real invariants surface. Resist the urge to wrap every primitive.

### Why `Data.Class` instead of raw `class`

Effect's `Data.Class` provides structural `Equal` and `Hash` for free, integrates with `Effect.match` and the rest of the Effect ecosystem, and avoids the boilerplate of hand-rolled equality. A raw `class` plus a schema-based factory also works but adds friction with Effect-flavored code paths in the application layer.

### Why no schema validation inside the value object

Schema parsing happens at the boundary — at the contract layer for inputs from the wire, or in `Email.fromString(s)` style factories. The value object itself stores already-validated data. Re-running validation on every construction is wasted work and duplicates the contract layer's job.

## Entity Documentation Pattern

Entities are documented in **markdown**, not implemented as classes.

### Where the docs live

`packages/domain/docs/entities/`, one file per entity. Entity-level docs live next to the package whose types they describe, so a developer looking at `@app/domain` types finds the rules in the same tree. (The docs site is reserved for cross-cutting architecture patterns.)

### Per-entity file structure

Each entity doc has six sections, in order:

1. **Purpose** — one paragraph: what this entity represents in the domain language, and what it does NOT represent. Link to the relevant feature docs.
2. **Field reference** — table of fields with constraints. For each field: type, nullable?, mutability, where the constraint is enforced (contract, domain factory, use case).
3. **Invariants** — bulleted rules that must hold for any valid instance. For each: which layer enforces it (contract validation, value object factory, application use case, DB constraint).
4. **State transitions** — if the entity has lifecycle states, the allowed transitions and the use cases that perform them.
5. **Common queries** — read patterns that recur across the application. Useful for both developers and AI agents.
6. **Examples** — minimal example payloads in JSON shape (what consumers see), plus one or two examples of broken state with the rule that catches it.

### Why markdown over class implementations

- **Team-shareable.** A non-technical product person can read invariants in plain text.
- **AI-navigable.** Agents read the docs and produce code that respects the rules.
- **Survives refactors.** A field rename in the schema does not invalidate the conceptual rules.
- **No serialization friction.** Entities ride the wire as JSON. Class instances are not JSON — you never reach for `toJson()` / `fromJson()` because you never produce class instances.

The cost is that compile-time enforcement of entity-level rules is replaced by runtime enforcement plus documentation. That is acceptable because the contract layer + value object factories already enforce most invariants at boundaries, and use cases enforce business rules in a typed way.

## Validation Layers

Every invariant is enforced **somewhere**, and the "somewhere" matters. Three layers, in order of cheapness:

| Layer                    | What it catches                                     | Tool                                 |
| ------------------------ | --------------------------------------------------- | ------------------------------------ |
| **Contract (schema)**    | Shape, type, basic constraints (length, range)      | `z.object`, `z.string().min(1)`      |
| **Value object**         | Primitive-level invariants (currency match, format) | `Money.add`, `InviteCode.fromString` |
| **Application use case** | Business rules across multiple entities             | Effect with `Data.TaggedError`       |

- **Shape and basic constraints — contract schema.** The contract package validates inputs from the wire; schema errors return as 400-shaped responses. Domain validation is not duplicated here.
- **Primitive invariants — value object factories.** `Money.fromMinorUnits(n, currency)` rejects negative amounts; `Email.fromString(s)` rejects malformed input. These factories return `Effect<VO, ValidationError>` so the use case composes them with the rest of its logic.
- **Business rules — application use cases.** "User must be a member to add a line item." "Remaining amount cannot go below zero." Enforced in `packages/application/src/{module}/{use-case}.ts` using typed errors.

## Effect Interaction

When the application layer uses Effect-TS, it interacts with this strategy as follows:

- **Use cases enforce invariants.** They check cross-entity rules and return `Effect<T, TaggedError>` for failure paths.
- **`Data.TaggedError` for typed errors.** One error class per business rule. The `_tag` string is what the consumer bridge switches on to map to RPC error codes.
- **No class entities.** Effect favors data + functions over OO. Plain types in the domain interoperate naturally; class instances do not.
- **`Data.Class` for value objects.** Cheap structural equality and Effect ergonomics.
- **`createAppLayer(db)` wires repos.** Repository constructors are synchronous and trivial.

See [ADR 0001: Effect for Application Use Cases](./decisions/adr-0001-effect-use-cases.md).

## What This Strategy Explicitly Does NOT Adopt

- **Heavyweight ORM-style entity classes.** No private fields with public getters, no `static create()` factory, no `validate()` method, no `getProps()` / `toJson()` ceremony. Too much overhead for the value; fights a data-first style.
- **Hand-rolled `Result` validation.** No `{ isValid: boolean; error?: string }`. Use `Effect` / `Either` instead.
- **Value objects for every primitive.** Only for primitives with real invariants. Branded `string` covers identity; schema covers shape.
- **Repositories per entity.** Prefer one repository per aggregate-shaped concern (e.g., an `OrderRepository` covering order + members + invites). Splitting per entity multiplies wiring without benefit.
- **Aggregates and transactional consistency boundaries.** Not modeled explicitly yet. Treat each use case as the consistency boundary. Revisit when concurrent updates surface real bugs.
- **Anti-corruption layers and bounded-context translation.** Not relevant while there is one bounded context. Revisit when a second context emerges.

## Trade-offs and Signals to Escalate

### What you trade

- Compile-time invariant enforcement on entity construction → runtime enforcement in use cases plus markdown docs.
- Rich entity behavior methods (`order.cancel()`) → use case functions (`cancelOrder(input)`).
- Class-based polymorphism → discriminated unions and `Effect.match`.

### Signals that you have outgrown this strategy

If any of these recur, escalate to richer DDD patterns:

- **Same invariant duplicated across multiple use cases.** Move the rule into a value object or a domain service.
- **Inconsistent state between two entities that always change together.** Model as an aggregate with a single root that owns both writes.
- **Concurrent update conflicts** that branded IDs and use-case-level checks cannot catch. Adopt optimistic concurrency at the repository layer.
- **A second bounded context with its own ubiquitous language.** Split `packages/domain` into per-context subpackages, and introduce anti-corruption layer translators at the boundary.
- **Use case files exceed ~150 lines and start growing private helpers.** Extract domain services or expose richer methods on value objects.

Until then, this strategy keeps the cost low.

## Related

- [Application Layer](./application-layer.md) — Layer-first package structure and dependency rules
- [ADR 0001: Effect for Application Use Cases](./decisions/adr-0001-effect-use-cases.md)
- [Domain Layer Contracts](./domain-layer-contracts.md) — Repository interfaces and entity types in the domain package
- [Domain Architecture Patterns](./domain-architecture-patterns.md) — Pure-DDD vs schema-only; this strategy is the reconciled middle path
