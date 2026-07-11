# Server Functions

_When to use TanStack Start server functions — theme/cookie work, and the boundary with an isomorphic API client._

## Purpose

Server functions live at `apps/web/src/server-functions/` and are the way the web app runs server-side logic without going through the backend API app. Each file exports exactly one `createServerFn` call and nothing else at runtime.

## When to use server functions vs the API client

An isomorphic typed API client at `apps/web/src/lib/api-client.ts` works in browser code AND in SSR contexts (route loaders), with cookie forwarding handled automatically. That covers all API data fetching, including SSR.

Server functions are reserved for the narrow set of cases where direct cookie/headers access on the **web** server matters and going through the API app is not the right boundary.

| Path                        | Where it runs              | Use for                                                                     |
| --------------------------- | -------------------------- | --------------------------------------------------------------------------- |
| API client (`api`/query)    | Browser AND SSR loaders    | All API data fetching, mutations, route prefetch                            |
| `createServerFn` (serverFn) | Server (web Worker)        | Theme cookie reads/writes, web-side cookie work that does not belong in API |

### Rule of thumb

- **Component or hook needs API data?** Use `apiQuery.x.y.queryOptions(...)` via a hook.
- **Mutation triggered by user action?** Use `apiQuery.x.y.mutationOptions(...)` via a hook.
- **Route loader prefetching API data?** Call `api.x.y(...)` directly from the loader.
- **Need to read or set a cookie on the web server (theme toggle, etc.)?** Server function.

> A common early decision is to treat the API client as "browser-only" and route SSR through server functions. This tends to be transitional — the isomorphic client eventually replaces it for all API data.

### Typical active server functions

`get-theme.ts`, `set-theme.ts` — theme management requires direct cookie access on the web server. New server functions are added only when something genuinely belongs on the web Worker (not on the API app).

## How it works

### One function per file

Each file must only export the `createServerFn` value at runtime. Any additional top-level runtime exports (helpers, constants, plain objects) can cause server-only infrastructure (e.g. the DB client package) to be pulled into the client bundle — a driver like `neon()` crashes on module load in the browser.

Shared server-side helpers belong in `apps/web/src/lib/server/`.

TypeScript `export type` declarations are safe because types are erased at compile time and do not affect the client bundle.

### Input validation

Server functions validate input with Zod via `.inputValidator((input: unknown) => schema.parse(input))`. Manual `if (!x) throw new Error(...)` checks are not used.

Schemas live in one of two places:

- **Domain schemas** (`packages/domain/src/schemas/`) — for schemas shared across web and API consumers
- **Inline schemas** — defined at the top of the server function file when the schema is used only there

Cross-field validations use `.superRefine()`.

### Auth and cookie work

```typescript
import { createServerFn } from "@tanstack/react-start";
import { getAuthSession } from "@/lib/auth/get-auth-session";

export const setThemeServerFn = createServerFn({ method: "POST" })
  .inputValidator(/* ... */)
  .handler(async (ctx) => {
    const session = await getAuthSession();
    // set cookie via getRequestHeaders / setHeaders / etc.
  });
```

`getAuthSession()` validates the session cookie against the DB directly and works in the web Worker even though the auth handler itself lives on the API app.

## Key files

| Path                                        | Role                                                         |
| ------------------------------------------- | ------------------------------------------------------------ |
| `apps/web/src/server-functions/`           | One file per `createServerFn` (theme + web-only cookie work) |
| `apps/web/src/lib/server/`                  | Shared server-only helpers (no `createServerFn` here)        |
| `apps/web/src/lib/auth/get-auth-session.ts` | Cookie-based session lookup                                  |
| `packages/domain/src/schemas/`             | Shared Zod input schemas                                     |

## Patterns

### Inline schema for one-off input

```typescript
const inputSchema = z.object({
  theme: z.enum(["light", "dark"]),
});

export const setThemeFn = createServerFn({ method: "POST" })
  .inputValidator((input: unknown) => inputSchema.parse(input))
  .handler(async (ctx) => {
    // ...
  });
```

### Domain schema import

```typescript
import { someInputSchema } from "@repo/domain/schemas";

export const myFn = createServerFn({ method: "POST" })
  .inputValidator((input: unknown) => someInputSchema.parse(input))
  .handler(/* ... */);
```

## Constraints

- One `createServerFn` per file; no additional runtime exports
- No `drizzle-orm` or DB schema imports — use repository classes from `@repo/infra-db/repositories` if a server function genuinely needs DB access
- Repositories are instantiated inline: `const repo = new EntityRepository(db)` — no DI container in server functions
- Don't reach for a server function when the work belongs on the API app — use the API client. Server functions are for web-only cookie work.

## Gotchas

- Any runtime export beyond the server function (even a plain `export const config = {}`) leaks the server-only DB client into the client bundle and causes a driver crash (`neon()` and similar) on load.
- `z.string().min(1)` accepts whitespace-only strings (length > 0) — chain `.trim()` before `.min(1)` for all user-facing text fields.
- `z.ZodIssueCode.custom` is deprecated in Zod v4 — use the string literal `"custom"` in `ctx.addIssue({ code: "custom", message: "..." })` inside `superRefine`.

## Related

- [Data Loading Patterns](./data-loading.md) — Isomorphic client used by hooks, components, and SSR loaders
