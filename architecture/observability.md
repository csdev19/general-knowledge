# Observability & Error Handling

_Error tracking, structured logging, and sanitization strategy across a web app and an HTTP API — users never see internal errors, everything is logged server-side._

## Philosophy

- Users **NEVER** see internal errors (SQL queries, stack traces, DB errors).
- All errors are logged server-side for debugging.
- The client shows generic "Something went wrong" messages.
- Real error details go to observability tools (platform logs, product analytics).

## Error Tracking Layers

### Layer 1: Platform Logs (Backend)

All server-side errors are logged as **structured JSON** via `console.error`. Log platforms that index JSON fields (e.g. Cloudflare Workers Logs) automatically extract each field from `console.error` output, making it searchable and filterable in the dashboard. This is why structured JSON is preferred over string concatenation — each field (`fn`, `error`, `stack`, `userId`, etc.) becomes a first-class filterable property with no additional configuration.

Both the web app and the API use this; no additional SDK is needed.

#### The `logError` Utility

Each app has a `logError` utility that standardizes structured error logging:

- **web**: `apps/web/src/lib/log.ts`
- **api**: `apps/api/src/utils/log.ts`

```ts
export function logError(fn: string, error: unknown, context?: Record<string, unknown>) {
  console.error(
    JSON.stringify({
      fn,
      error: error instanceof Error ? error.message : String(error),
      stack: error instanceof Error ? error.stack : undefined,
      ...context,
    }),
  );
}
```

- `fn` — identifies the origin of the error (see naming convention below)
- `error` — safely extracts `.message` from Error instances, or stringifies unknown values
- `context` — optional key-value pairs (e.g. `userId`, `resourceId`) spread into the JSON for richer filtering

Use cases in `packages/application/` use inline `JSON.stringify` instead of a shared utility, since application-layer packages do not import from app-level code.

#### Naming Convention for `fn`

| Origin                     | Format               | Example                   |
| -------------------------- | -------------------- | ------------------------- |
| Server functions (web)     | `"functionName"`     | `"getResources"`          |
| API endpoints (api)        | `"api:handlerName"`  | `"api:createResource"`    |
| Use cases (application)     | `"useCase:name"`     | `"useCase:createResource"`|
| Error boundary (frontend)  | `"ErrorFallback"`    | `"ErrorFallback"`         |
| Global API handler         | `"api:unhandled"`    | `"api:unhandled"`         |

#### Examples

```ts
// Server function (web) — uses the logError utility
import { logError } from "@/lib/log";
logError("getResources", error, { userId: session?.user?.id });

// API endpoint (api) — uses the logError utility
import { logError } from "@/utils/log";
logError("api:createResource", error, { userId: context.user.id });

// Use case (packages/application/) — inline JSON.stringify (no app-level imports)
console.error(
  JSON.stringify({
    fn: "useCase:createResource",
    error: error instanceof Error ? error.message : String(error),
    stack: error instanceof Error ? error.stack : undefined,
  }),
);

// Global error handler (api)
console.error(
  JSON.stringify({
    fn: "api:unhandled",
    error: err.message,
    stack: err.stack,
    path: c.req.path,
    method: c.req.method,
  }),
);
```

### Layer 2: Product Analytics Error Tracking (Frontend)

A product-analytics SDK (e.g. PostHog) captures errors on the client through multiple mechanisms:

| Mechanism                   | What it catches                                                        | Configuration                                                                                           |
| --------------------------- | --------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| `capture_exceptions` config | Unhandled errors and unhandled promise rejections via `window.onerror` | `capture_unhandled_errors: true`, `capture_unhandled_rejections: true`, `capture_console_errors: false` |
| Error boundary integration  | React rendering crashes                                                | Wraps the app in `__root.tsx` with `source: "react_error_boundary"`                                     |
| `captureException()`        | Caught errors in mutation callbacks                                    | Called manually with a `{ source: "flow_name" }` property                                               |
| `QueryCache.onError`        | Failed data-fetching queries                                           | Global handler in `query-client.ts` with query-key context                                              |

Tag every exception with an `app` property (e.g. `app: "web"`) for filtering in the analytics dashboard.

### Layer 3: Application-Level Error Handling

- **Server functions**: try/catch wrapper returns `{ data: null, error: { message: "generic" } }`.
- **API endpoints**: a global `onError` handler returns sanitized JSON with HTTP 500.
- **API routes**: a typed error (e.g. `ORPCError`) for expected errors (NOT_FOUND, BAD_REQUEST, FORBIDDEN).
- **Use cases**: a `Result` (or typed-error) channel with safe messages.
- **Frontend**: an `ErrorFallback` shows a generic message with a sign-out option.

## Error Response Convention

### Server Functions (web)

```ts
// Shape: { data: T | null, error: { message: string } | null }

// Success
return { data: result, error: null };

// Business error (user-safe message)
return { data: null, error: { message: "Unauthorized" } };

// Unhandled error (generic message — never expose error.message)
catch (error) {
  logError("functionName", error, { userId: session?.user?.id });
  return { data: null, error: { message: "Something went wrong. Please try again." } };
}
```

### API Endpoints (api)

Same `{ data, error }` shape wrapped in HTTP status codes:

```ts
// Expected errors — use a typed error with safe messages
throw new ORPCError("NOT_FOUND", { message: "Not found" });
throw new ORPCError("FORBIDDEN", { message: "Not authorized" });
throw new ORPCError("BAD_REQUEST", { message: "Invalid operation" });

// Unexpected errors — the global handler sanitizes automatically
app.onError((err, c) => {
  console.error(JSON.stringify({
    fn: "api:unhandled", error: err.message, stack: err.stack,
    path: c.req.path, method: c.req.method,
  }));
  return c.json({ data: null, error: { message: "Internal server error" } }, 500);
});

// Not-found routes
app.notFound((c) => c.json({ data: null, error: { message: "Not found" } }, 404));
```

## Error Sanitization Rules

1. **NEVER** pass `error.message` from a caught exception to a client response.
2. **NEVER** show SQL queries, stack traces, or DB error messages to users.
3. Log the real error with `logError()` or structured `console.error(JSON.stringify(...))` so the platform indexes the fields.
4. Use typed error codes for programmatic handling; generic messages for display.
5. Use-case errors should be business-level ("Not a member"), not technical ("FK constraint violation").

## Frontend Error Display

The `ErrorFallback` component handles catastrophic errors:

- Shows "Something went wrong" with a "Please try again or contact support" message.
- Provides a sign-out button that redirects to the login route.
- Logs the real error via `console.error` for the platform.
- Never displays `error.message` or stack traces to the user.

For non-catastrophic errors:

- Mutation failures: `toast.error()` with a generic message + `captureException()` for tracking.
- Query failures: `QueryCache.onError` shows a toast with a retry action + `captureException()`.

## Best Practices

1. **Every try/catch must log** — silent catches hide bugs.
2. **Always use structured JSON logging** — `logError("fn", error, { userId })` instead of `console.error('[fn]', error)`, so the platform can index fields.
3. **Follow the `fn` naming convention** — `"functionName"`, `"api:handlerName"`, `"useCase:name"`.
4. **Include context in logs** — pass relevant IDs (`userId`, `resourceId`) in the `context` parameter for faster debugging.
5. **Separate user feedback from error tracking** — `toast.error()` for UX, `logError` + `captureException` for debugging (both required in every mutation `onError`).
6. **Never trust `error.message` for display** — errors from the DB, network, or libraries contain internal details.
7. **Test error paths** — verify error pages show generic messages, not internal details.
8. **Use typed error codes for expected API errors** — proper HTTP status codes and safe messages.

## Error Flow Diagram

```
User Action
  |
  v
Frontend (web)
  |-- React rendering error --> Error Boundary --> ErrorFallback (generic message)
  |-- Mutation error --> onError callback --> toast.error() + captureException()
  |-- Query error --> QueryCache.onError --> toast.error() + captureException()
  |
  v
Server Function / API Handler
  |-- Business error --> { data: null, error: { message: "User-safe message" } }
  |-- Unhandled error --> logError("fn", error, { userId }) --> Platform Logs
  |                   --> { data: null, error: { message: "Something went wrong" } }
  |
  v
Platform Logs
  |-- All console.error JSON output indexed automatically
  |-- Filterable by structured fields: fn, error, userId, resourceId, etc.
```

## A Maturity Roadmap

A useful progression for building this out incrementally:

**Foundation (do first)**

- A `withErrorHandling` wrapper that centralizes try/catch, structured logging, and the `{ data, error }` envelope.
- An `AppError` class with typed error codes (UNAUTHORIZED, FORBIDDEN, NOT_FOUND, RATE_LIMITED, VALIDATION, CONFLICT, INTERNAL) and an `Errors` factory for ergonomic creation.
- A `requireAuth()` helper replacing inline session checks.
- Request timing (`duration_ms`) and auto-extracted context IDs on every log; success logging (not just errors); request-timing middleware on the API.

**Richer observability**

- Expand error codes for specific failure modes.
- Export traces to a free-tier OTLP destination.
- Saved queries for error rate by `fn`; alert triggers for error-rate spikes (e.g. >5% of requests returning 500).

**Middleware & correlation**

- Observability middleware for timing/context; auth middleware replacing per-handler `requireAuth()`; chain `observability -> auth -> handler`.
- Generate a unique `X-Request-ID` per client request, thread it through server functions and API handlers, and include it in every log and captured exception — enabling a single user action to be traced across frontend and backend.

**Health checks**

- A `/health` endpoint that verifies DB connectivity with a lightweight query (`SELECT 1`) and returns a degraded status if the DB is unreachable; monitor with an external uptime service.

**Effect migration (long-term)**

- Replace `AppError` with `Data.TaggedError`; define `Layer`s for the DB client, auth session, and repositories.
- Replace the `withErrorHandling` wrapper with Effect's runtime (handles the envelope automatically) and `logError` with `Effect.withSpan` + OpenTelemetry for native tracing. Function signatures then explicitly declare all possible errors and dependencies in their types.

## Related

- [Security Hardening](./security-hardening.md) — Authorization checks and data-safety patterns.
