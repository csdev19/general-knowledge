# Application Services Layer

_Implementing the Application layer with Use Cases, Commands, Queries, DTOs, and mappers on top of a rich domain model._

The **Application Layer** (also called Application Services or Use Cases layer) orchestrates domain logic to fulfill specific user actions. It sits between the presentation layer (HTTP/UI) and the domain layer.

---

## 🎯 Purpose of Application Layer

The Application Layer:

- ✅ **Orchestrates** domain objects to complete a task
- ✅ **Validates** input from external sources
- ✅ **Manages** transactions and persistence
- ✅ **Publishes** domain events
- ✅ **Transforms** domain models to DTOs

It does **NOT**:

- ❌ Contain business rules (those go in domain)
- ❌ Access the database directly (uses repositories)
- ❌ Handle HTTP concerns (that's infrastructure)

---

## 📦 Application Layer Structure

```bash
packages/your-context/application/
├── use-cases/                    # Commands (writes)
│   ├── create-entity/
│   │   ├── create-entity.command.ts
│   │   ├── create-entity.handler.ts
│   │   ├── create-entity.validator.ts
│   │   └── index.ts
│   ├── update-status/
│   │   ├── update-status.command.ts
│   │   ├── update-status.handler.ts
│   │   └── index.ts
│   ├── add-comment/
│   └── delete-entity/
│
├── queries/                      # Queries (reads)
│   ├── get-entity/
│   │   ├── get-entity.query.ts
│   │   ├── get-entity.handler.ts
│   │   └── index.ts
│   ├── list-entities/
│   └── get-entity-stats/
│
├── dto/                          # Data Transfer Objects
│   ├── entity.dto.ts
│   ├── create-entity.dto.ts
│   ├── update-entity.dto.ts
│   └── index.ts
│
├── services/                     # Application services (if needed)
│   └── entity.service.ts
│
├── events/                       # Application event handlers
│   └── entity-created.handler.ts
│
└── index.ts                      # Barrel exports
```

---

## 🔀 CQRS Pattern (Commands and Queries)

**CQRS** (Command Query Responsibility Segregation) separates writes from reads:

### Commands (Write Operations)

**Change system state, no return value (or just an ID)**

```typescript
// use-cases/create-entity/create-entity.command.ts

/**
 * Command to create a new entity.
 * Commands represent user intentions to change system state.
 */
export class CreateEntityCommand {
  constructor(
    public readonly userId: string,
    public readonly title: string,
    public readonly description: string,
    public readonly priority?: number,
    public readonly category?: string,
    public readonly tags?: string[],
  ) {}
}
```

**Command Handler:**

```typescript
// use-cases/create-entity/create-entity.handler.ts

import type { IEntityRepository } from "@app/your-context/domain/repositories";
import type { IEventPublisher } from "@app/shared-kernel/events";
import { Entity } from "@app/your-context/domain/entities";
import { Priority } from "@app/your-context/domain/value-objects";
import type { CreateEntityCommand } from "./create-entity.command";
import { Result, tryCatch } from "@app/shared-kernel/result";

export class CreateEntityHandler {
  constructor(
    private readonly entityRepo: IEntityRepository,
    private readonly eventPublisher: IEventPublisher,
  ) {}

  async execute(command: CreateEntityCommand): Promise<Result<string, Error>> {
    // 1. Business rules already validated by schema at API boundary.

    // 2. Create domain entity with business logic
    const priority = command.priority
      ? Priority.create({ level: command.priority, category: command.category || "default" })
      : null;

    const entity = Entity.create({
      userId: command.userId,
      title: command.title,
      description: command.description,
      priority,
    });

    // 3. Persist using repository
    const saveResult = await tryCatch(this.entityRepo.save(entity));
    if (saveResult.error) return saveResult;

    // 4. Publish domain events
    await this.eventPublisher.publishAll(entity.getDomainEvents());

    // 5. Return success with ID
    return { data: entity.id, error: null };
  }
}
```

---

### Queries (Read Operations)

**Read system state, no side effects**

```typescript
// queries/get-entity/get-entity.query.ts

/** Query to get a single entity. Queries fetch data without changing state. */
export class GetEntityQuery {
  constructor(
    public readonly entityId: string,
    public readonly userId: string,
  ) {}
}
```

**Query Handler:**

```typescript
// queries/get-entity/get-entity.handler.ts

import type { IEntityRepository } from "@app/your-context/domain/repositories";
import type { GetEntityQuery } from "./get-entity.query";
import type { EntityDTO } from "../../dto/entity.dto";
import { EntityMapper } from "../../mappers/entity-dto.mapper";
import { Result } from "@app/shared-kernel/result";

export class GetEntityHandler {
  constructor(private readonly entityRepo: IEntityRepository) {}

  async execute(query: GetEntityQuery): Promise<Result<EntityDTO, Error>> {
    const entity = await this.entityRepo.findById(query.entityId, query.userId);
    if (!entity) {
      return { data: null, error: new Error("Entity not found") };
    }

    // Transform domain entity to DTO
    const dto = EntityMapper.toDTO(entity);
    return { data: dto, error: null };
  }
}
```

---

## 📋 Data Transfer Objects (DTOs)

DTOs are simple data containers for transferring data between layers. They are:

- Plain objects (no methods)
- Framework-agnostic
- Easy to serialize/deserialize
- Different from domain entities

```typescript
// dto/entity.dto.ts

import type { EntityStatus } from "@app/your-context/domain";

export interface EntityDTO {
  id: string;
  userId: string;
  title: string;
  description: string;
  status: EntityStatus;
  priority: number | null;
  category: string;
  createdAt: Date;
  updatedAt: Date;
}

export interface CreateEntityDTO {
  title: string;
  description: string;
  priority?: number;
  category?: string;
}

export interface UpdateEntityDTO {
  title?: string;
  description?: string;
  status?: EntityStatus;
  priority?: number;
  category?: string;
}

export interface EntityListItemDTO {
  id: string;
  title: string;
  description: string;
  status: EntityStatus;
  createdAt: Date;
}
```

---

## 🗺️ DTO Mappers

Mappers convert between domain entities and DTOs:

```typescript
// mappers/entity-dto.mapper.ts

import type { Entity } from "@app/your-context/domain/entities";
import type { EntityDTO, EntityListItemDTO } from "../dto/entity.dto";

export class EntityDTOMapper {
  static toDTO(entity: Entity): EntityDTO {
    return {
      id: entity.id,
      userId: entity.userId,
      title: entity.title,
      description: entity.description,
      status: entity.status.value,
      priority: entity.priority?.level ?? null,
      category: entity.category ?? "default",
      createdAt: entity.createdAt,
      updatedAt: entity.updatedAt,
    };
  }

  static toListItemDTO(entity: Entity): EntityListItemDTO {
    return {
      id: entity.id,
      title: entity.title,
      description: entity.description,
      status: entity.status.value,
      createdAt: entity.createdAt,
    };
  }

  static toListDTOs(entities: Entity[]): EntityListItemDTO[] {
    return entities.map((entity) => this.toListItemDTO(entity));
  }
}
```

---

## 🏗️ Application Service (Alternative Pattern)

Instead of separate Commands/Queries, you can group related operations in an Application Service class:

```typescript
// services/entity.service.ts

import type { IEntityRepository } from "@app/your-context/domain/repositories";
import type { IEventPublisher } from "@app/shared-kernel/events";
import { Entity } from "@app/your-context/domain/entities";
import type { CreateEntityDTO, UpdateEntityDTO, EntityDTO } from "../dto";
import { EntityDTOMapper } from "../mappers/entity-dto.mapper";
import { Result, tryCatch } from "@app/shared-kernel/result";

export class EntityService {
  constructor(
    private readonly entityRepo: IEntityRepository,
    private readonly eventPublisher: IEventPublisher,
  ) {}

  async create(userId: string, dto: CreateEntityDTO): Promise<Result<string, Error>> {
    const entity = Entity.create({ userId, ...dto });
    const result = await tryCatch(this.entityRepo.save(entity));
    if (result.error) return result;
    await this.eventPublisher.publishAll(entity.getDomainEvents());
    return { data: entity.id, error: null };
  }

  async update(id: string, userId: string, dto: UpdateEntityDTO): Promise<Result<EntityDTO, Error>> {
    const entity = await this.entityRepo.findById(id, userId);
    if (!entity) return { data: null, error: new Error("Entity not found") };

    // Update entity (business logic lives in the domain)
    if (dto.title) entity.updateTitle(dto.title);
    if (dto.status) entity.updateStatus(dto.status);

    const result = await tryCatch(this.entityRepo.update(entity));
    if (result.error) return { data: null, error: result.error };
    return { data: EntityDTOMapper.toDTO(entity), error: null };
  }
}
```

---

## 🔌 Wiring in Infrastructure (HTTP Layer)

The application layer is called from HTTP routes with dependency injection:

```typescript
// apps/api/src/modules/your-context/entity.router.ts

import { implement, ORPCError } from "@orpc/server";
import { entityContract } from "../../contract/entity.contract";
import { CreateEntityHandler, CreateEntityCommand } from "@app/application";
import { createDatabaseClient } from "@app/infra-db/client";
import { EntityRepository } from "@app/infra-db/repositories";
import { authMiddleware } from "../../middleware/auth";
import { env } from "../../env";

const db = createDatabaseClient(env.DATABASE_URL);
// The handler depends only on the domain repository interface.
const createEntityHandler = new CreateEntityHandler(new EntityRepository(db), eventPublisher);

const impl = implement(entityContract).$context<{ headers: Headers }>();

export const entityRouter = impl.router({
  create: impl.create.use(authMiddleware).handler(async ({ input, context }) => {
    const command = new CreateEntityCommand(
      context.user.id, input.title, input.description, input.priority, input.category, input.tags,
    );
    const result = await createEntityHandler.execute(command);
    if (result.error) throw new ORPCError("BAD_REQUEST", { message: result.error.message });
    return { data: { id: result.data }, error: null };
  }),
});
```

---

## 🧪 Testing Application Layer

The application layer is easy to test with mocks because it depends only on interfaces:

```typescript
import { describe, it, expect, beforeEach, vi } from "vitest";
import { CreateEntityHandler } from "../create-entity.handler";
import { CreateEntityCommand } from "../create-entity.command";
import type { IEntityRepository } from "@app/your-context/domain/repositories";
import type { IEventPublisher } from "@app/shared-kernel/events";

describe("CreateEntityHandler", () => {
  let handler: CreateEntityHandler;
  let mockRepo: IEntityRepository;
  let mockEventPublisher: IEventPublisher;

  beforeEach(() => {
    mockRepo = { save: vi.fn().mockResolvedValue(undefined), findById: vi.fn() };
    mockEventPublisher = { publishAll: vi.fn().mockResolvedValue(undefined) };
    handler = new CreateEntityHandler(mockRepo, mockEventPublisher);
  });

  it("should create entity successfully", async () => {
    const command = new CreateEntityCommand("user-123", "Example Entity", "A sample description", 1, "default");
    const result = await handler.execute(command);
    expect(result.error).toBeNull();
    expect(result.data).toBeDefined();
    expect(mockRepo.save).toHaveBeenCalledOnce();
    expect(mockEventPublisher.publishAll).toHaveBeenCalledOnce();
  });

  it("should return error when repository fails", async () => {
    mockRepo.save = vi.fn().mockRejectedValue(new Error("DB Error"));
    const command = new CreateEntityCommand("user-123", "Example Entity", "A sample description");
    const result = await handler.execute(command);
    expect(result.error).toBeDefined();
    expect(result.data).toBeNull();
  });
});
```

---

## 📐 Application Layer Patterns

### Pattern 1: Command/Query Handlers (Recommended)

```
use-cases/create-entity/
  - command.ts      ← Command definition
  - handler.ts      ← Command handler
  - validator.ts    ← Optional validation
```

**Benefits:** Single Responsibility, easy to test, clear separation, scalable.

### Pattern 2: Application Service Class

```
services/
  - entity.service.ts  ← All operations in one class
```

**Benefits:** grouped related operations, less boilerplate, familiar pattern.
**Drawbacks:** can become a large God class with mixed responsibilities.

### Pattern 3: Feature Folders (Vertical Slices)

```
features/create-entity/
  - command.ts
  - handler.ts
  - validator.ts
  - dto.ts
  - route.ts
```

**Benefits:** everything related to a feature in one place, easy to find, can be moved to a separate package.

---

## 🎯 Best Practices

### 1. Keep the Application Layer Thin

```typescript
// ❌ Bad — business rule in application layer
async create(dto: CreateDTO) {
  if (dto.priority < 0) throw new Error("Priority must be positive");
  // ...
}

// ✅ Good — business rule lives in the domain
async create(dto: CreateDTO) {
  const entity = Entity.create(dto);  // Validation inside the entity
  await this.repo.save(entity);
}
```

### 2. Use DTOs for External Communication

```typescript
// ✅ Good — return DTO, not the domain entity
async getById(id: string): Promise<EntityDTO> {
  const entity = await this.repo.findById(id);
  return EntityDTOMapper.toDTO(entity);
}

// ❌ Bad — domain entity leaks out of the layer
async getById(id: string): Promise<Entity> {
  return await this.repo.findById(id);
}
```

### 3. Use the Result Pattern for Error Handling

```typescript
// ✅ Explicit error handling instead of throwing
async execute(command: Command): Promise<Result<string, Error>> {
  return await tryCatch(this.repo.save(entity));
}
```

### 4. Publish Domain Events

```typescript
async execute(command: Command) {
  const entity = Entity.create(command);
  await this.repo.save(entity);
  await this.eventPublisher.publishAll(entity.getDomainEvents());
}
```

---

## 🔗 Related

- [Bounded Contexts Complete Guide](./bounded-contexts-complete-guide.md)
- [Repository Pattern](./repository-pattern.md)
- [Domain Architecture Patterns](./domain-architecture-patterns.md)
