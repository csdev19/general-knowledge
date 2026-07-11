# Repository Pattern Architecture

_Splitting repositories into a domain-owned interface and an infrastructure-owned implementation, following Dependency Inversion — plus a before/after migration guide._

This guide explains how to implement the Repository pattern following Domain-Driven Design principles.

---

## 🎯 Core Principle

**Repositories are split between two packages:**

1. **Interface (Contract)** → `packages/domain/` — _what_ the repository can do
2. **Implementation** → `packages/database/` (or `infra-db/`) — _how_ it actually does it

This follows the **Dependency Inversion Principle** and keeps the domain clean and independent of infrastructure concerns.

---

## 📦 Package Structure

```bash
packages/
  domain/
    your-context/
      entities/
        entity.entity.ts             # Rich domain model (or plain type)
      repositories/
        entity.repository.ts         # ← INTERFACE (contract)
      value-objects/
        amount.ts
        display-name.ts

  database/
    your-context/
      repositories/
        entity.repository.drizzle.ts # ← IMPLEMENTATION
      mappers/
        entity.mapper.ts             # Maps DB rows ↔ domain entities
    schema/
      entity.schema.ts               # ORM table definition

  application/
    your-context/
      use-cases/
        get-entities.service.ts      # Orchestrates business flow
        create-entity.service.ts
```

---

## 🔑 Why This Separation?

### Dependency Flow

```
┌─────────────────────────────────────────────┐
│  packages/application/  (Use Cases)         │
│    ↓ depends on                             │
│  packages/domain/  (Interface + Entities)   │
└─────────────────────────────────────────────┘
                    ↑ implements
┌─────────────────────────────────────────────┐
│  packages/database/  (Implementation + ORM) │
└─────────────────────────────────────────────┘
```

### Benefits

| Benefit                    | Why It Matters                                     |
| -------------------------- | -------------------------------------------------- |
| **Separation of Concerns** | Domain logic is independent of the database        |
| **Testability**            | Can test use cases without a real DB               |
| **Flexibility**            | Can swap the ORM without touching the domain       |
| **Clean Dependencies**     | Domain doesn't depend on infrastructure            |
| **Rich Domain Model**      | Entities contain business logic, not just data     |

---

## 📝 Implementation Example

### 1. Repository Interface (Domain Layer)

```typescript
// packages/domain/your-context/repositories/entity.repository.ts
import type { Entity } from "../entities/entity.entity";
import type { PaginationParams, PaginatedResult } from "@app/domain/types";

export interface IEntityRepository {
  findById(id: string, userId: string): Promise<Entity | null>;
  findPaginated(userId: string, params: PaginationParams): Promise<PaginatedResult<Entity>>;
  save(entity: Entity): Promise<void>;
  delete(id: string, userId: string): Promise<void>;
}
```

**Key points:** works with domain entities (not raw DB rows), returns rich domain objects, no SQL/ORM references, pure TypeScript.

### 2. Repository Implementation (Database Layer)

```typescript
// packages/database/your-context/repositories/entity.repository.drizzle.ts
import { eq, and, desc, sql, isNull } from "drizzle-orm";
import { entityTable } from "../../schema";
import type { DatabaseClient } from "@app/infra-db/client";
import type { IEntityRepository } from "@app/domain/your-context/repositories";
import type { Entity } from "@app/domain/your-context/entities";
import type { PaginationParams, PaginatedResult } from "@app/domain/types";
import { EntityMapper } from "../mappers/entity.mapper";

export class EntityRepositoryDrizzle implements IEntityRepository {
  constructor(private readonly db: DatabaseClient) {}

  async findById(id: string, userId: string): Promise<Entity | null> {
    const row = await this.db.query.entityTable.findFirst({
      where: and(
        eq(entityTable.id, id),
        eq(entityTable.userId, userId),
        isNull(entityTable.deletedAt),
      ),
    });
    return row ? EntityMapper.toDomain(row) : null;
  }

  async findPaginated(userId: string, params: PaginationParams): Promise<PaginatedResult<Entity>> {
    const offset = (params.page - 1) * params.limit;
    const [entities, countResult] = await Promise.all([
      this.db.select().from(entityTable)
        .where(and(eq(entityTable.userId, userId), isNull(entityTable.deletedAt)))
        .orderBy(desc(entityTable.updatedAt))
        .limit(params.limit).offset(offset),
      this.db.select({ count: sql<number>`count(*)` }).from(entityTable)
        .where(and(eq(entityTable.userId, userId), isNull(entityTable.deletedAt))),
    ]);
    const total = Number(countResult[0]?.count || 0);
    return {
      data: entities.map((row) => EntityMapper.toDomain(row)),
      total, page: params.page, limit: params.limit,
      totalPages: Math.ceil(total / params.limit),
    };
  }

  async save(entity: Entity): Promise<void> {
    await this.db.insert(entityTable).values(EntityMapper.toPersistence(entity));
  }

  async delete(id: string, userId: string): Promise<void> {
    await this.db.update(entityTable)
      .set({ deletedAt: new Date() })
      .where(and(eq(entityTable.id, id), eq(entityTable.userId, userId)));
  }
}
```

**Key points:** implements the domain interface, uses the ORM for DB operations, uses mappers to convert rows ↔ entities, encapsulates all SQL/ORM logic.

### 3. Mapper (Database Layer)

```typescript
// packages/database/your-context/mappers/entity.mapper.ts
import type { Entity } from "@app/domain/your-context/entities";

export class EntityMapper {
  static toDomain(row: any): Entity {
    return {
      id: row.id, userId: row.userId, name: row.name, title: row.title,
      status: row.status, description: row.description,
      createdAt: row.createdAt, updatedAt: row.updatedAt,
    };
  }

  static toPersistence(entity: Entity): any {
    return {
      id: entity.id, userId: entity.userId, name: entity.name, title: entity.title,
      status: entity.status, description: entity.description,
      createdAt: entity.createdAt, updatedAt: entity.updatedAt,
    };
  }
}
```

### 4. Application Service (Use Case)

```typescript
// packages/application/your-context/use-cases/get-entities.service.ts
import type { IEntityRepository } from "@app/domain/your-context/repositories";
import type { PaginationParams, PaginatedResult } from "@app/domain/types";
import type { Entity } from "@app/domain/your-context/entities";

export class GetEntities {
  constructor(private readonly entityRepo: IEntityRepository) {}

  async execute(userId: string, params: PaginationParams): Promise<PaginatedResult<Entity>> {
    // Validate business rules here if needed (permissions, filters, etc.)
    return await this.entityRepo.findPaginated(userId, params);
  }
}
```

**Key points:** depends only on the interface, orchestrates business flow, testable with mock repositories.

### 5. Controller/Route (Infrastructure)

```typescript
// apps/api/src/modules/your-context/entity.router.ts
import { implement } from "@orpc/server";
import { entityContract } from "../../contract/entity.contract";
import { createDatabaseClient } from "@app/infra-db/client";
import { EntityRepository } from "@app/infra-db/repositories";
import { getEntities } from "@app/application";
import { authMiddleware } from "../../middleware/auth";
import { env } from "../../env";

const db = createDatabaseClient(env.DATABASE_URL);
const repo = new EntityRepository(db);
const impl = implement(entityContract).$context<{ headers: Headers }>();

export const entityRouter = impl.router({
  list: impl.list.use(authMiddleware).handler(async ({ input, context }) => {
    const result = await getEntities({ repo, userId: context.user.id, pagination: input });
    if (result.error) throw result.error;
    return { data: result.data.data, error: null, meta: { pagination: result.data.meta } };
  }),
});
```

---

## 🚫 What NOT to Do

**❌ Direct queries in routes**

```typescript
// DON'T — querying the DB straight from the route handler
list: impl.list.use(authMiddleware).handler(async ({ context }) => {
  const entities = await db.select().from(entityTable).where(eq(entityTable.userId, context.user.id));
  return { data: entities, error: null };
}),
```

**❌ A "queries" folder in the database package** that exports raw query functions called directly from routes.

**❌ Repository returning raw DB data** (`Promise<any>` of a raw row) instead of a domain entity.

---

## ✅ Rules to Follow

| Rule                                       | Why                                |
| ------------------------------------------ | ---------------------------------- |
| Repository interfaces in `domain/`         | Domain defines what it needs       |
| Repository implementations in `database/`  | Infrastructure provides how        |
| Always return domain entities              | Never expose raw DB data           |
| Use mappers for conversions                | Keep domain and DB layers separate |
| Application services orchestrate           | Routes stay thin                   |
| No ORM references in domain                | Domain stays pure                  |

---

## 🧪 Testing Benefits

With this architecture, use cases are trivial to test with an in-memory mock:

```typescript
class MockEntityRepository implements IEntityRepository {
  private entities: Entity[] = [];
  async findPaginated(userId: string, params: PaginationParams) {
    return { data: this.entities, total: this.entities.length, page: 1, limit: 10, totalPages: 1 };
  }
  // ... other methods
}

const mockRepo = new MockEntityRepository();
const useCase = new GetEntities(mockRepo);
const result = await useCase.execute("user-123", { page: 1, limit: 10 });
```

---

## 🔄 Refactoring an Existing Codebase

If a feature currently uses direct queries, this is the before/after and the migration steps.

### Before (❌ Anti-pattern)

```
packages/db/src/queries/entity.ts     ← Direct query functions
  └─ findPaginatedEntities()          ← Called directly from routes

apps/api/src/routes/entities.ts
  └─ Direct SQL/ORM queries in route handlers
```

Problems: domain logic mixed with infrastructure, no separation of concerns, hard to test, tight coupling to the ORM, raw queries in routes.

### After (✅ Clean Architecture)

```
packages/domain/src/repositories/entity.repository.ts          ← Interface (contract)
packages/database/src/repositories/entity.repository.drizzle.ts ← Implementation
packages/database/src/mappers/entity.mapper.ts                 ← DB ↔ Domain conversion
apps/api/src/routes/entities.ts                                ← Uses repository interface
```

### Route: Before vs After

```typescript
// Before — direct database queries in the route
const entities = await db.select().from(entityTable)
  .where(and(eq(entityTable.userId, user.id), isNull(entityTable.deletedAt)))
  .orderBy(desc(entityTable.updatedAt))
  .limit(pagination.limit).offset(pagination.offset);

// After — clean, abstracted repository call
const result = await entityRepo.findPaginated(user.id, {
  page: query.page || 1,
  limit: query.limit || 10,
});
```

### Package Exports

```json
// packages/domain/package.json
{ "exports": { "./repositories": "./src/repositories/index.ts" } }

// packages/database/package.json
{ "exports": {
    "./repositories": "./src/repositories/index.ts",
    "./mappers": "./src/mappers/index.ts"
} }
```

### Migration Steps

1. **Create the repository interface** in `packages/domain/`.
2. **Create the repository implementation** in `packages/database/`.
3. **Create a mapper** for DB ↔ domain conversion.
4. **Update routes** to use the repository.
5. **Delete old query files.**
6. **Write tests** with mock repositories.

### Impact

| Aspect                 | Before       | After          |
| ---------------------- | ------------ | -------------- |
| Separation of Concerns | ❌ Mixed     | ✅ Clear       |
| Testability            | ❌ Hard      | ✅ Easy        |
| Domain Independence    | ❌ Coupled   | ✅ Independent |
| Reusability            | ❌ Low       | ✅ High        |
| Maintainability        | ❌ Difficult | ✅ Simple      |

---

## 💡 Key Takeaway

> **"From the domain's perspective, it's like having a list of objects, even though they're in a database."**

The Repository pattern abstracts away data access concerns, allowing the domain to stay clean, testable, and independent of infrastructure decisions.

> **Note:** starting with **plain entity types** (data containers) and no explicit mappers is a valid MVP — infra repos can return objects that structurally match the domain types, and TypeScript catches drift at the `implements` boundary. Introduce rich entity classes and explicit mappers when behavior and DB/domain shape divergence actually appear. See [Domain Layer Contracts](./domain-layer-contracts.md).

## 🔗 Related

- [Application Services Layer](./application-services-layer.md)
- [Bounded Contexts Complete Guide](./bounded-contexts-complete-guide.md)
- [Domain Layer Contracts](./domain-layer-contracts.md)
