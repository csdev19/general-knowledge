# Domain Layer Contracts

_Extracting repository interfaces and plain entity types into the domain package so infrastructure implements domain-defined contracts (hexagonal architecture)._

## What It Does

Extracts domain-level abstractions out of the infrastructure layer into `packages/domain/`. This includes:

- **Repository interfaces** (contracts) that define what queries the domain needs.
- **Entity types** that represent the domain model independent of the database.
- **Domain constants/utilities** (e.g. an `generateInviteCode()` helper) that encode domain knowledge, moved out of infrastructure.

Infrastructure repositories then **implement** these interfaces. The domain declares the port; infra provides the adapter.

## Structure

```bash
packages/domain/src/
  repositories/
    order.repository.ts          # IOrderRepository interface
    member.repository.ts         # IMemberRepository interface
    line-item.repository.ts      # ILineItemRepository interface
    payment.repository.ts        # IPaymentRepository interface
    ...
    index.ts                     # Barrel export for all interfaces
  types/
    entities.ts                  # Plain domain entity types + computed value objects
    index.ts                     # Re-exports entity types
  constants/
    invite-code.ts               # generateInviteCode() — domain knowledge
    index.ts

packages/infra-db/src/
  repositories/*.ts              # Implement the domain interfaces
```

Each domain repository interface defines the query contract that consumers (server functions, API routes) depend on, while the infra-db repository classes provide the concrete (e.g. Drizzle) implementation.

```typescript
// packages/domain/src/repositories/order.repository.ts
import type { Order } from "../types/entities";

export interface IOrderRepository {
  findById(id: string): Promise<Order | null>;
  findByInviteCode(code: string): Promise<Order | null>;
  create(order: Order): Promise<Order>;
  // ... query contract the domain needs, expressed in domain terms
}
```

```typescript
// packages/infra-db/src/repositories/order.repository.ts
import type { IOrderRepository } from "@app/domain/repositories";
import type { Order } from "@app/domain/types";

export class OrderRepository implements IOrderRepository {
  constructor(private readonly db: Database) {}
  async findById(id: string): Promise<Order | null> {
    /* Drizzle query returning an object shaped like Order */
  }
}
```

## Domain Model as Plain Types

Entity types are **plain TypeScript types** (no classes, no methods) representing the domain model without any database or ORM coupling.

- Interfaces return domain entity types (e.g. `Order`) rather than raw query results.
- Computed value objects (e.g. a `MemberBalance` that is calculated, never stored) exist only as return types from a repository interface, not as persisted entities.
- Domain constants live as `as const` enum-likes (statuses, types, methods, frequencies).

## Decisions

| Decision                                        | Why                                                                                     | Alternatives Considered                          |
| ----------------------------------------------- | --------------------------------------------------------------------------------------- | ------------------------------------------------ |
| Plain types, not classes                        | Entity types are data containers; no behavior yet. Classes add ceremony with no value.  | Class-based entities with methods                |
| Interfaces prefixed `I` (e.g. `IOrderRepository`) | Follows a consistent repository-interface naming convention.                          | Suffix `Contract` or `Port`                      |
| Domain-concept helpers in domain constants      | Rules like invite-code char set and length are domain knowledge, not infrastructure.    | Keep the helper inside the infra repository      |
| No mappers yet                                  | Infra repos return objects matching domain types directly; mappers add overhead now.    | Explicit mapper classes for DB row → domain type |

## Gotchas & Warnings

- Infra repos return objects that **structurally match** the domain entity types but are not explicitly mapped. If the ORM schema drifts from the domain types, TypeScript catches it at the `implements` boundary.
- Computed value objects (like a `MemberBalance`) are **not stored entities** — they only exist as a return type from a repository. Don't try to persist them.
- When you introduce behavior or real DB/domain shape divergence, graduate from plain types to rich entity classes with explicit mappers — see [Repository Pattern](./repository-pattern.md).

## Testing

```bash
# Type-check to verify interfaces are correctly implemented at the `implements` boundary
bunx tsc --noEmit -p apps/api/tsconfig.json
```

## Related

- [Repository Pattern](./repository-pattern.md) — Full DDD repository pattern guide (the target pattern)
- [Domain Modeling Strategy](./domain-modeling-strategy.md) — Branded IDs, value objects, invariant enforcement
- [Application Layer](./application-layer.md) — How consumers wire domain interfaces to infra implementations
