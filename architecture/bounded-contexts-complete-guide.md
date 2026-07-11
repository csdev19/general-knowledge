# Bounded Contexts — Complete Architecture Guide

_Organizing packages per bounded context with full DDD layering, integration patterns, and a migration path from a single context to many._

This is a complete guide to implementing Bounded Contexts in a monorepo, covering package structure, layers, integration patterns, and worked examples.

---

## 🎯 What is a Bounded Context?

A **Bounded Context** is an explicit boundary within which a particular domain model applies. It's a logical division of your business domain where:

- 🔹 Terms have specific meanings
- 🔹 Models are self-contained
- 🔹 Rules are enforced
- 🔹 Teams can work independently

### Example: "User" in Different Contexts

```typescript
// Core Context
class ContextUser {
  createEntity()
  trackEntityStatus()
  uploadDocument()
}

// Analytics Context
class AnalyticsUser {
  getActivityMetrics()
  calculateEngagement()
}

// Billing Context
class SubscriptionUser {
  subscribeToPlan()
  processPayment()
  checkUsageLimit()
}
```

**Same real-world person, different meaning in each context.**

---

## 📦 Complete Package Structure Per Context

```bash
app/                                 # Monorepo root
├── apps/                            # Applications
│   ├── web/                         # Frontend app
│   ├── api/                         # Backend API
│   └── documentation/               # Documentation
│
├── packages/                        # Shared packages
│   ├── shared-kernel/               # ← Shared primitives
│   │   └── src/
│   │       ├── api.ts               # ApiResponse, Pagination
│   │       ├── currency.ts          # Currency constants
│   │       ├── result.ts            # Result<T, E> pattern
│   │       ├── utils.ts             # TypeScript utilities
│   │       └── index.ts
│   │
│   ├── your-context/                # ← Core Bounded Context
│   │   ├── domain/                  # Domain layer
│   │   │   ├── entities/
│   │   │   ├── value-objects/
│   │   │   ├── repositories/        # Interfaces only
│   │   │   ├── services/            # Domain services
│   │   │   ├── events/              # Domain events
│   │   │   └── constants/
│   │   ├── application/             # Use cases, queries, DTOs
│   │   ├── infrastructure/          # Repo implementations, mappers
│   │   ├── package.json
│   │   └── tsconfig.json
│   │
│   ├── analytics/                   # ← Analytics Bounded Context
│   │   ├── domain/
│   │   ├── application/
│   │   └── infrastructure/
│   │
│   ├── billing/                     # ← Billing Bounded Context
│   │   ├── domain/
│   │   ├── application/
│   │   └── infrastructure/
│   │       └── payment-providers/   # e.g. a payment gateway adapter
│   │
│   ├── integration/                 # ← Integration layer (Context → Context)
│   │   ├── your-context-api/        # Public API (DTOs) for the core context
│   │   ├── analytics-api/           # Public API for analytics
│   │   └── package.json
│   │
│   ├── database/                    # ← Shared database infrastructure
│   │   ├── schema/                  # Table definitions
│   │   └── client/
│   │
│   └── web-ui/                      # ← Shared UI components
│
└── package.json                     # Root package.json
```

---

## 🏗️ Layers Within Each Context

Each bounded context follows **Clean Architecture** with distinct layers.

### 1. Domain Layer (Core Business Logic)

**Goes here:** entities with behavior, value objects with invariants, domain services, repository interfaces, domain events, domain exceptions.
**Does NOT go here:** database queries, HTTP requests, framework dependencies, UI logic.

```typescript
// packages/your-context/domain/entities/entity.entity.ts
import { EntityStatus } from "../value-objects/entity-status";
import { Priority } from "../value-objects/priority";
import { EntityCreatedEvent } from "../events/entity-created.event";

export class Entity {
  private constructor(
    public readonly id: string,
    private title: string,
    private status: EntityStatus,
    private priority: Priority | null,
    private comments: Comment[],
  ) {}

  static create(data: { title: string; priority?: Priority }): Entity {
    const entity = new Entity(
      crypto.randomUUID(), data.title, EntityStatus.draft(), data.priority ?? null, [],
    );
    entity.raise(new EntityCreatedEvent(entity.id, data.title));
    return entity;
  }

  updateStatus(newStatus: EntityStatus): void {
    if (!this.status.canTransitionTo(newStatus)) {
      throw new InvalidStatusTransitionError(this.status, newStatus);
    }
    this.status = newStatus;
  }

  addComment(comment: Comment): void {
    if (!this.isActive()) throw new Error("Cannot add comment to closed entity");
    this.comments.push(comment);
  }

  isActive(): boolean {
    return this.status.isActive();
  }
}
```

### 2. Application Layer (Use Cases / Orchestration)

**Goes here:** use case handlers (Commands/Queries), application services, DTOs, input validation, transaction orchestration.
**Does NOT go here:** business rules (domain), database implementation, HTTP routing.

```typescript
// packages/your-context/application/use-cases/create-entity/create-entity.handler.ts
import type { IEntityRepository } from "@app/your-context/domain";
import { Entity } from "@app/your-context/domain";
import { Priority } from "@app/your-context/domain/value-objects";
import type { CreateEntityCommand } from "./create-entity.command";

export class CreateEntityHandler {
  constructor(
    private readonly entityRepo: IEntityRepository,
    private readonly eventPublisher: IEventPublisher,
  ) {}

  async execute(command: CreateEntityCommand): Promise<string> {
    const priority = command.priority ? Priority.create(command.priority) : null;
    const entity = Entity.create({ title: command.title, priority });
    await this.entityRepo.save(entity);
    await this.eventPublisher.publish(entity.getDomainEvents());
    return entity.id;
  }
}
```

### 3. Infrastructure Layer (Technical Implementation)

**Goes here:** repository implementations, ORM/database code, external API adapters, file system, email/SMS services.
**Does NOT go here:** business logic, use case orchestration.

```typescript
// packages/your-context/infrastructure/persistence/repositories/entity.repository.ts
import type { IEntityRepository, Entity } from "@app/your-context/domain";
import { EntityMapper } from "../mappers/entity.mapper";
import { db } from "@app/database/client";

export class EntityRepositoryImpl implements IEntityRepository {
  async save(entity: Entity): Promise<void> {
    await db.insert(entityTable).values(EntityMapper.toPersistence(entity));
  }

  async findById(id: string): Promise<Entity | null> {
    const row = await db.query.entityTable.findFirst({ where: eq(entityTable.id, id) });
    return row ? EntityMapper.toDomain(row) : null;
  }
}
```

---

## 🔗 Context Integration Patterns

### Pattern 1: Published Language (DTOs)

Each context exposes a **Public API** for other contexts to consume:

```typescript
// packages/integration/your-context-api/dto/entity.dto.ts
export interface EntityDTO {
  id: string;
  title: string;
  status: string;  // Simple string, not the rich domain object
  priority: number | null;
  category: string;
  createdAt: Date;
}

export interface EntitySummaryDTO {
  totalEntities: number;
  activeEntities: number;
  successRate: number;
}
```

The analytics context consumes the core context's summary via this DTO API rather than reaching into its domain.

### Pattern 2: Domain Events (Asynchronous Integration)

Contexts communicate through events without direct coupling:

```typescript
// packages/your-context/domain/events/entity-created.event.ts
export class EntityCreatedEvent {
  constructor(
    public readonly entityId: string,
    public readonly title: string,
    public readonly userId: string,
    public readonly occurredAt: Date = new Date(),
  ) {}
}

// packages/analytics/application/event-handlers/entity-created.handler.ts
export class OnEntityCreated {
  constructor(private readonly metricsRepo: IMetricsRepository) {}
  async handle(event: EntityCreatedEvent): Promise<void> {
    await this.metricsRepo.incrementTotalEntities(event.userId);
  }
}
```

### Pattern 3: Anti-Corruption Layer (ACL)

Protect your context from external models by translating at the boundary:

```typescript
// packages/analytics/infrastructure/anti-corruption/context-adapter.ts
import type { EntityDTO } from "@app/integration/your-context-api";
import { EntityMetric } from "@app/analytics/domain";

export class ContextAdapter {
  toDomain(dto: EntityDTO): EntityMetric {
    return new EntityMetric({
      entityId: dto.id,
      wasSuccessful: dto.status === "completed",
      duration: this.calculateDuration(dto),
    });
  }
}
```

---

## 📋 package.json Per Context

```json
// packages/your-context/package.json
{
  "name": "@app/your-context",
  "type": "module",
  "exports": {
    "./domain": "./domain/index.ts",
    "./application": "./application/index.ts",
    "./infrastructure": "./infrastructure/index.ts"
  },
  "dependencies": {
    "@app/shared-kernel": "workspace:*",
    "@app/database": "workspace:*"
  }
}
```

```json
// packages/analytics/package.json — depends on the integration API, not the core context directly
{
  "name": "@app/analytics",
  "type": "module",
  "dependencies": {
    "@app/shared-kernel": "workspace:*",
    "@app/integration": "workspace:*"
  }
}
```

---

## 🎯 Dependency Rules

### ✅ Allowed

```
Application Layer  ── can use ──▶  Domain Layer
Infrastructure Layer  ── implements ──▶  Domain Layer
All layers  ── can use ──▶  Shared Kernel
```

```typescript
import { Entity } from "@app/your-context/domain";                 // Application uses Domain ✅
import type { IEntityRepository } from "@app/your-context/domain"; // Infra implements Domain ✅
import { Currency } from "@app/shared-kernel";                     // All use Shared Kernel ✅
```

### ❌ Forbidden

```
Domain → Application       ❌
Domain → Infrastructure    ❌
Context A → Context B (directly)  ❌
```

### ✅ Context-to-Context Communication

Always go through integration packages:

```typescript
import type { EntityDTO } from "@app/integration/your-context-api";          // ✅
import type { EntityCreatedEvent } from "@app/integration/your-context-api/events"; // ✅
```

---

## 🚀 Worked Example: One Action Spanning Contexts

**Scenario:** a user creates an entity. Three contexts react.

```typescript
// 1. Core context handles creation and publishes an event
export class CreateEntityHandler {
  async execute(command: CreateEntityCommand): Promise<string> {
    const entity = Entity.create(command);
    await this.repo.save(entity);
    await this.eventBus.publish(new EntityCreatedEvent(entity.id, command.userId));
    return entity.id;
  }
}

// 2. Analytics context reacts
export class OnEntityCreated {
  async handle(event: EntityCreatedEvent): Promise<void> {
    await this.metricsRepo.incrementEntityCount(event.userId);
  }
}

// 3. Billing context reacts — checks usage limits
export class OnEntityCreatedForBilling {
  async handle(event: EntityCreatedEvent): Promise<void> {
    const subscription = await this.subscriptionRepo.findByUserId(event.userId);
    if (subscription.hasExceededLimit()) {
      await this.notificationService.sendLimitWarning(event.userId);
    }
  }
}
```

---

## 📊 Context Mapping

```
┌──────────────────────────────────────────────────────────────┐
│                     Shared Kernel                            │
│  (Currency, Result, ApiResponse, Pagination)                 │
└──────────────────────────────────────────────────────────────┘
                            ↑ uses
        ┌───────────────────┼───────────────────┐
        │                   │                   │
┌───────▼────────┐  ┌───────▼─────────┐  ┌───────▼────────┐
│  Core Context  │  │Analytics Context│  │ Billing Context│
└────────┬───────┘  └────────┬────────┘  └────────┬───────┘
         │    publishes       │    listens        │
         │    events          │    to events      │
         └──────────►─────────┴──────◄────────────┘
                            │
                    ┌───────▼────────┐
                    │  Integration   │
                    │  (Event Bus)   │
                    └────────────────┘
```

---

## 💡 Best Practices

**1. Keep contexts small and focused.**

```bash
✅ @app/your-context (single responsibility)
❌ @app/your-context-analytics-billing (mixed concerns)
```

**2. Use explicit boundaries** — export only what other layers/contexts should consume; don't leak infrastructure.

**3. Version your integration APIs** (`your-context-api/v1`, `.../v2`) so consumers can migrate gradually.

**4. Test contexts independently** — domain unit tests need no database.

---

## 🎯 Migration Strategy

Start simple and split as a second domain emerges.

```
Phase 1 — Single Context
  packages/domain/        ← All business logic
  packages/database/      ← All infrastructure

Phase 2 — Extract Shared Kernel
  packages/shared-kernel/ ← Generic primitives
  packages/domain/
  packages/database/

Phase 3 — Add Second Context
  packages/shared-kernel/
  packages/your-context/  ← Renamed from domain
  packages/analytics/     ← New context
  packages/database/

Phase 4 — Full Multi-Context
  packages/shared-kernel/
  packages/your-context/
  packages/analytics/
  packages/billing/
  packages/integration/   ← Context APIs
  packages/database/
```

---

## 🔗 Related

- [Shared Kernel](./shared-kernel.md)
- [Repository Pattern](./repository-pattern.md)
- [Application Services Layer](./application-services-layer.md)
- [Domain Modeling Strategy](./domain-modeling-strategy.md)
