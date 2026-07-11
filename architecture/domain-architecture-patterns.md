# Domain Architecture Patterns

_Entity-Based (DDD) vs Schema-Based domain modeling — trade-offs, when to use each, and a gradual migration path between them._

This document explains two fundamental approaches to modeling domain logic: **Entity-Based (DDD)** and **Schema-Based**. The choice significantly impacts code organization, maintainability, and complexity.

## Architecture Comparison

### Entity-Based Architecture (Domain-Driven Design)

**Philosophy:** domain models are rich objects with both data and behavior. Business logic lives within the entities themselves.

```typescript
export class Reservation {
  private readonly _id: UniqueIdVO;
  private readonly _status: ReservationStatusVO;
  private readonly _startTime: AvailableFromVO;

  public cancel(at: Date): Reservation {
    // Business logic here
    return Reservation.create({
      ...this.toJson(),
      status: "CANCELLED",
      isInactive: at.toISOString(),
      updatedAt: at,
    });
  }

  public confirm(at: Date): Reservation {
    // Business logic here
  }
}
```

### Schema-Based Architecture (Data Transfer Pattern)

**Philosophy:** domain models are validated data structures. Business logic lives in service layers or functions.

```typescript
// Validation schemas
export const entityBaseSchema = z.object({
  id: z.string().uuid(),
  name: z.string().min(1),
  status: z.enum(["active", "rejected", "cancelled", "completed"]),
  amount: z.number().min(0).nullable(),
  currency: z.enum(["USD", "EUR"]),
  createdAt: z.date(),
  updatedAt: z.date(),
});

// Derived schemas
export const createEntitySchema = entityBaseSchema
  .pick({ name: true, status: true })
  .extend({
    amount: z.number().min(0).optional(),
    currency: z.enum(["USD", "EUR"]).optional(),
  });

export const updateEntitySchema = createEntitySchema;

// Type inference
export type Entity = z.infer<typeof entityBaseSchema>;
export type CreateEntity = z.infer<typeof createEntitySchema>;
export type UpdateEntity = z.infer<typeof updateEntitySchema>;
```

---

## Detailed Comparison

### Entity-Based (DDD)

**✅ Advantages**

1. **Rich domain model** — business logic is encapsulated within entities; methods like `cancel()` / `confirm()` express domain operations.
2. **Type safety with value objects** — `UniqueIdVO`, `ReservationStatusVO` prevent primitive obsession; validation happens at the value object level.
3. **Encapsulation** — private fields with public getters; you cannot construct an invalid entity; invariants are always enforced.
4. **Testability** — unit test business logic independently of infrastructure.
5. **DDD benefits** — ubiquitous language reflected in code; better for complex business rules.

**❌ Disadvantages**

1. **Complexity overhead** — more boilerplate, steeper learning curve, extra value-object layers.
2. **Serialization issues** — classes don't serialize naturally to JSON; require `toJson()`; hydration challenges on the client.
3. **Frontend integration friction** — React prefers plain objects; you can't easily store class instances in state.
4. **Over-engineering risk** — may be overkill for simple CRUD.
5. **Shared type challenges** — entities are backend-centric; the frontend usually needs separate DTOs.

### Schema-Based

**✅ Advantages**

1. **Simplicity** — minimal boilerplate, easy to understand and maintain.
2. **API-first design** — direct JSON validation; works with REST/RPC/GraphQL; no serialization complexity.
3. **Frontend-friendly** — plain objects work everywhere; no hydration issues.
4. **Easy derivation** — `.pick()`, `.omit()`, `.partial()` for variants with automatic type inference.
5. **Shared validation** — same schemas in frontend and backend; single source of truth.
6. **Perfect for CRUD** — most web apps are CRUD-heavy; schemas suffice.

**❌ Disadvantages**

1. **No behavior encapsulation** — validation only; business rules scatter into services.
2. **Anemic domain model** — just data structures; an anti-pattern for complex domains.
3. **Less semantic type safety** — primitives (`string`) instead of value objects (`Email`, `UserId`).
4. **Business logic duplication risk** — the same logic may exist in multiple places.
5. **Scales poorly for complex domains** — as rules grow it can become procedural spaghetti.

---

## When to Use Each Approach

### Use Entity-Based (DDD) when:

- **Complex business rules** — state machines, workflows, multi-step processes with invariants, where domain logic _is_ the core business value.
- **Long-term projects** — multiple teams, an evolving domain, maintainability is critical.
- **Domain expertise required** — close collaboration with stakeholders, ubiquitous language matters, complex bounded contexts.

Example domains: booking/reservation systems, financial systems with complex rules, e-commerce with inventory and pricing logic, healthcare with regulatory requirements.

### Use Schema-Based when:

- **Simple CRUD operations** — validation is the primary concern; few complex rules.
- **API-first applications** — data transfer is the main purpose; frontend and backend share types.
- **Fast iteration needed** — startups, MVPs, prototypes, small teams.

Example domains: content management systems, dashboards and analytics, admin panels, basic CRUD applications.

---

## Migration Path: From Schemas to Entities

If an application grows in complexity, migrate gradually.

### Phase 1: Pure Schemas

```typescript
export const entityBaseSchema = z.object({
  id: z.string().uuid(),
  name: z.string().min(1),
  status: z.enum(["active", "rejected", "cancelled", "completed"]),
});
export type Entity = z.infer<typeof entityBaseSchema>;
```

### Phase 2: Add Domain Functions

```typescript
export function canTransitionStatus(from: EntityStatus, to: EntityStatus): boolean {
  const allowedTransitions = {
    active: ["rejected", "cancelled", "completed"],
    rejected: [],
    cancelled: [],
    completed: [],
  };
  return allowedTransitions[from]?.includes(to) ?? false;
}

export function calculateDaysSinceLastUpdate(entity: Entity): number {
  const now = new Date();
  const updated = new Date(entity.updatedAt);
  return Math.floor((now.getTime() - updated.getTime()) / (1000 * 60 * 60 * 24));
}

export function isStale(entity: Entity): boolean {
  return calculateDaysSinceLastUpdate(entity) > 30;
}
```

### Phase 3: Introduce Value Objects

```typescript
export class EntityStatusVO {
  private constructor(private readonly value: EntityStatus) {}

  public static create(value: string): EntityStatusVO {
    if (!isValidStatus(value)) throw new Error(`Invalid status: ${value}`);
    return new EntityStatusVO(value as EntityStatus);
  }

  public canTransitionTo(target: EntityStatus): boolean {
    // Encapsulate business logic
  }

  public getValue(): EntityStatus {
    return this.value;
  }
}
```

### Phase 4: Full Entity Model

```typescript
export class Entity {
  private readonly _id: UniqueIdVO;
  private readonly _status: EntityStatusVO;
  private readonly _name: string;

  private constructor(props: EntityProps) {
    this._id = UniqueIdVO.create(props.id);
    this._status = EntityStatusVO.create(props.status);
    this._name = props.name;
  }

  public static create(props: EntityProps): Entity {
    const validated = entityBaseSchema.parse(props);
    return new Entity(validated);
  }

  public reject(reason?: string): Entity {
    if (!this._status.canTransitionTo("rejected")) {
      throw new Error("Cannot reject from current status");
    }
    return Entity.create({ ...this.toJson(), status: "rejected", updatedAt: new Date() });
  }

  public toJson() {
    return { id: this._id.toString(), name: this._name, status: this._status.getValue() };
  }
}
```

---

## Best Practices for Schema-Based Architecture

1. **Organize schemas by entity** — one file per domain entity, related schemas grouped.
2. **Use schema derivation** — a base schema as the source of truth; derive create/update/partial with `.pick()`, `.omit()`.
3. **Place business logic in services** — keep schemas focused on validation.
4. **Export inferred types** — consistent naming: `BaseSchema`, `CreateSchema`, `UpdateSchema`.
5. **Share schemas across layers** — backend validation and frontend forms use the same schema.

### Signals to Consider Migrating to Entities

- ⚠️ Same business logic duplicated in multiple places
- ⚠️ Complex state transitions with many rules
- ⚠️ Domain invariants being violated
- ⚠️ Stakeholders requesting more complex features
- ⚠️ Team spending time coordinating business logic changes
- ⚠️ Difficulty testing business rules

---

## Conclusion

Both approaches are legitimate. A CRUD-focused, API-first application with simple state transitions and a need for fast iteration is well served by the **schema-based approach** — it keeps the codebase simple while staying flexible. If the application evolves to require complex business logic, gradually introduce domain functions, value objects, and eventually full entities.

For a reconciled middle path between these two poles (contract types as wire truth, branded IDs, value objects only for high-leverage primitives, invariants enforced in use cases), see [Domain Modeling Strategy](./domain-modeling-strategy.md).

## References

- [Domain-Driven Design by Eric Evans](https://www.domainlanguage.com/ddd/)
- [Anemic Domain Model by Martin Fowler](https://martinfowler.com/bliki/AnemicDomainModel.html)
- [Value Objects Pattern](https://martinfowler.com/bliki/ValueObject.html)
