# Application Layer & Package Architecture

_Layer-first architecture for sharing business logic across multiple server-side consumers (an HTTP API + framework server functions) while keeping the domain client-safe._

## The Problem

A monorepo often serves multiple consumers that need the same business logic:

| Consumer                     | How it calls business logic                   | Example              |
| ---------------------------- | --------------------------------------------- | -------------------- |
| **Web app** (server fns)     | Server function — runs on the server, no HTTP hop | Direct function call |
| **HTTP API**                 | HTTP routes — for external/mobile clients     | REST/RPC endpoint    |
| **Cron jobs / workers**      | Direct execution                              | Scheduled task       |

If a use case lived inside `apps/api/src/modules/`, the web app couldn't reuse it without making an HTTP call to itself. Duplicating the logic across consumers leads to drift and bugs. So business logic lives in a **shared package**, and each consumer wires it to its own infrastructure.

## The Architecture

A **layer-first** package structure where each package is an architectural layer, not a bounded context:

```
packages/domain/        <-- Pure. Client-safe. Constants, schemas, types, interfaces.
packages/application/   <-- Use cases. Server-only. Depends on domain interfaces.
packages/infra-db/      <-- Infrastructure. Server-only. Implements domain interfaces.

apps/web/               <-- Web app: server fn -> application -> infra-db
apps/api/               <-- HTTP API: route handler -> application -> infra-db
```

### Dependency Rule

The dependency arrow **always points inward** toward domain. Strict and non-negotiable:

```
domain        <-- depends on NOTHING (leaf package, client-safe)
application   <-- depends on domain ONLY (server-only)
infra-db      <-- depends on domain ONLY (server-only)
apps/*        <-- wire application + infra-db together
```

- `domain` never imports from `application` or `infra-db`
- `application` never imports from `infra-db` (uses domain interfaces instead)
- `infra-db` never imports from `application`
- Only the app layer (route handlers, server fns, workers) wires concrete implementations

### Why Layer-First, Not Vertical Bounded Contexts

You could organize by bounded context (one package per context with all layers inside). Layer-first is preferable when:

1. **Client safety is structural.** Any client surface (browser bundle, desktop renderer, mobile app) imports `@app/domain` — a package that _cannot_ contain infrastructure by design. With vertical slicing, a client would have to avoid `@app/your-context/application` — a subpath in the same package. One wrong import bundles your DB code into the client.
2. **Package boundary > subpath boundary.** A separate package is a harder boundary than a subpath export. You can't accidentally import what isn't in the dependency tree.
3. **Less overhead.** Three packages (domain, application, infra-db) instead of N packages per context, each with its own `package.json`, `tsconfig`, and build config.
4. **Cross-context sharing is trivial.** A type from another context is a relative import within the same package.

When bounded contexts grow, they become **folders within each layer package** (`packages/application/src/orders/`, a future `packages/application/src/billing/`, etc.) — see [Bounded Contexts](./bounded-contexts-complete-guide.md).

## Package Details

### `@app/domain` — Pure, Client-Safe

Contains only constants, validation schemas (e.g. Zod), TypeScript types (`ApiResponse`, `Result`, inferred schema types), and repository interfaces (e.g. `IEntityRepository` — erased at runtime).

**Zero dependencies on infrastructure.** No I/O, no side effects, no DB, no HTTP, no `node:*`. That purity is what makes it safe to import from any client. The schemas double as **contracts**: server validation and client-side form validation use the same schema, so the client sends exactly what the server expects.

### `@app/application` — Server-Only Use Cases

Use case functions that orchestrate business logic. Each one accepts **domain interfaces** as dependencies (not concrete implementations), takes **input validated by domain schemas**, and returns a domain type — `Result<T>` for explicit error handling, or the value directly for trivial passthroughs. **Zero infrastructure imports.**

Example — `listEntities` (`packages/application/src/entities/list-entities.ts`):

```typescript
import type { IEntityRepository } from "@app/domain/repositories";
import type { EntityBase } from "@app/domain/schemas";
import type { Result, PaginationQuery, PaginatedResult } from "@app/domain/types";

export async function listEntities(params: {
  repo: IEntityRepository;
  userId: string;
  pagination: PaginationQuery;
}): Promise<Result<PaginatedResult<EntityBase>>> {
  const { repo, userId, pagination } = params;
  const page = pagination.page ?? 1;
  const limit = pagination.limit ?? 5;
  const offset = (page - 1) * limit;

  try {
    const result = await repo.findAllByUserIdPaginated(userId, limit, offset);
    return {
      data: {
        data: result.data,
        meta: { page, limit, total: result.total, totalPages: Math.ceil(result.total / limit) },
      },
      error: null,
    };
  } catch (error) {
    return { data: null, error: error as Error };
  }
}
```

Notice: the function doesn't know whether it runs inside an HTTP route handler, a server function, or a cron job. It doesn't care — the caller provides the repo.

### `@app/infra-db` — Infrastructure

Concrete implementations: ORM schema definitions, repository implementations (`EntityRepository implements IEntityRepository`), mappers (DB row ↔ domain type), and the DB client factory.

## How Consumers Wire It Together

Each consumer wires the same use case to its own infrastructure. Both of these call `listEntities`:

**HTTP API route** (`apps/api/src/modules/entity/entity.router.ts`):

```typescript
import { implement } from "@orpc/server";
import { entityContract } from "../../contract/entity.contract";
import { createDatabaseClient } from "@app/infra-db/client";
import { EntityRepository } from "@app/infra-db/repositories";
import { listEntities } from "@app/application";
import { authMiddleware } from "../../middleware/auth";
import { env } from "../../env";

const db = createDatabaseClient(env.DATABASE_URL);
const repo = new EntityRepository(db);
const impl = implement(entityContract).$context<{ headers: Headers }>();

export const entityRouter = impl.router({
  list: impl.list.use(authMiddleware).handler(async ({ input, context }) => {
    const result = await listEntities({ repo, userId: context.user.id, pagination: input });
    if (result.error) throw result.error;
    return { data: result.data.data, error: null, meta: { pagination: result.data.meta } };
  }),
});
```

**Web app server function** (`apps/web/src/server-functions/list-entities.ts`) — no HTTP hop, calls the use case in-process:

```typescript
import { createServerFn } from "@tanstack/react-start";
import { paginationQuerySchema } from "@app/domain/schemas";
import { createDatabaseClient } from "@app/infra-db/client";
import { EntityRepository } from "@app/infra-db/repositories";
import { listEntities as listEntitiesUseCase } from "@app/application";
import { getAuthSession } from "@/lib/auth/get-auth-session";
import { env } from "@/env/server";

export const listEntities = createServerFn({ method: "GET" })
  .inputValidator((input) => paginationQuerySchema.parse(input))
  .handler(async (ctx) => {
    const session = await getAuthSession();
    if (!session) return { data: null, error: { message: "Unauthorized" } };

    const repo = new EntityRepository(createDatabaseClient(env.DATABASE_URL));
    const result = await listEntitiesUseCase({ repo, userId: session.user.id, pagination: ctx.data });
    if (result.error) return { data: null, error: { message: "Failed to list entities" } };

    return { data: result.data.data, error: null, meta: { pagination: result.data.meta } };
  });
```

Same use case, same repository, same input schema — two different transport layers. That is the entire point of the application package.

## The Contract Pattern

Validation schemas in `@app/domain/schemas` are the **contract** between a client and the application layer:

- **Input contract:** `createEntitySchema` / `CreateEntity` (inferred type)
- **Output contract:** `EntityBase` (type inferred from `entityBaseSchema`)
- **Error contract:** `ApiResponse<T>` with `{ data, error, meta }`

A client validates its form with the same schema the server validates the body with. Add a field to the schema and both update simultaneously — no drift.

## Client-Safe Domain

Any shipped client bundle is extractable, so the boundary matters:

- **Safe to import in a client** (`@app/domain`): constants, schemas, types/interfaces (erased at runtime), pure helpers with no I/O.
- **Never import in a client**: `@app/infra-db` (DB client, connection strings, ORM schemas), `@app/application` (server-only use cases), server-side auth config.

The boundary is enforced structurally by the package graph (a client only depends on `@app/domain`) and by the package import rules. A client that needs to add a repo call must go through the API, never import infra directly.

## When to Extract a Use Case

Extract into `@app/application` when **multiple consumers** need the same logic (web server fn + API route), when business logic goes beyond simple CRUD (defaults, transitions, orchestration, side effects), or when a handler exceeds ~15 lines of business logic.

Keep it inline in the handler when it's a trivial passthrough to the repository, only one consumer needs it, and there's no orchestration. (A `createEntity` that is just `repository.create({ ...data, userId })` stays inline — precisely because it has no orchestration yet.)

## Summary

| Layer          | Package                   | Client-safe? | Contains                                 |
| -------------- | ------------------------- | :----------: | ---------------------------------------- |
| Domain         | `@app/domain`             |     Yes      | Constants, schemas, types, interfaces    |
| Application    | `@app/application`        |      No      | Use cases, business-logic orchestration  |
| Infrastructure | `@app/infra-db`           |      No      | ORM repos, mappers, DB client            |
| Consumers      | `apps/web`, `apps/api`    |     N/A      | Wiring: route/server fn + repo + use case |
