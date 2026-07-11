# Error Handling Overhaul

_Centralized error handling with a `withErrorHandling` wrapper, typed `AppError` errors, and structured JSON logging._

## What It Does

Prevents internal error details (SQL queries, stack traces, DB errors) from ever reaching the user. Replaces inline try/catch in every server function with a centralized wrapper that handles error classification, structured logging for the hosting platform, and response envelope shaping. Users see generic "Something went wrong" messages while full error details go to the platform's log stream (e.g. Cloudflare Workers Logs).

> Motivation: a production incident where internal SQL and DB error details leaked into user-facing responses.

## Implementation

### Files Changed

| Path                                     | Purpose                                                                                   |
| ---------------------------------------- | ----------------------------------------------------------------------------------------- |
| `apps/web/src/lib/server/errors.ts`      | `AppError` class + `Errors` factory with 7 typed error codes                              |
| `apps/web/src/lib/server/with-error-handling.ts` | Centralized wrapper: try/catch, timing, context extraction, structured logging, envelope |
| `apps/web/src/lib/auth/require-auth.ts`  | Auth helper that throws `Errors.unauthorized()` instead of returning null                  |
| `apps/web/src/lib/log.ts`                | `logError()` utility — structured JSON via `console.error` for platform log indexing       |
| `apps/api/src/utils/log.ts`              | Same `logError()` utility for the API app                                                  |
| `apps/api/src/index.ts`                  | Global `app.onError()` + `app.notFound()` handlers                                         |
| `apps/api/src/modules/<resource>/<resource>.router.ts` | All endpoints sanitized with `logError` + generic messages                        |
| `apps/web/src/routes/__root.tsx`         | ErrorFallback: generic message + sign-out header, never shows `error.message`               |
| `apps/web/src/lib/query-client.ts`       | QueryCache global `onError` sanitized                                                      |
| `packages/application/src/**`            | Use cases wrapped with try/catch + structured JSON logging                                 |
| `apps/web/src/server-functions/**`       | All handler files refactored to use `withErrorHandling`                                     |

### Domain Model

N/A — infrastructure concern. `AppError` is an application-level error class, not a domain entity.

### Key Logic

Every server function handler is wrapped with `withErrorHandling("fnName", handler)`. The wrapper measures request duration via `performance.now()`, auto-extracts context IDs (e.g. `entityId`, `ownerId`) from input args, and catches all errors. `AppError` instances are mapped to user-safe messages by error code. Unknown errors get a generic message. Both success and error paths emit structured JSON logs that the hosting platform's log stream indexes automatically. Handlers throw `Errors.forbidden()`, `Errors.notFound()`, etc. instead of returning error objects, and return raw data instead of `{ data, error }` envelopes.

## Decisions

| Decision                                                                | Why                                                                                                  | Alternatives Considered                                                                        |
| ----------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| Higher-order wrapper function over framework middleware                 | The frontend framework's middleware had an open bug where it didn't reliably catch handler errors    | Middleware (future phase), inline try/catch (status quo)                                        |
| 7 error codes, not domain-specific codes                                | HTTP-level categories (FORBIDDEN, NOT_FOUND) cover all cases. Specifics go in the message string      | Fine-grained codes like `ENTITY_NOT_FOUND` (adds complexity without clear benefit at this scale) |
| `Errors.forbidden(msg)` uses same message for internal and user display | Business-level messages like "Not a member" are safe and informative for users                        | Generic "You don't have permission" (loses useful context for users)                            |
| Auto-extract context from input args                                    | Avoids requiring every handler to manually pass IDs to the wrapper                                     | Manual context param (more flexible but more boilerplate)                                        |
| Success logging disabled (commented out)                                | Excessive log volume in prod — every page load triggered multiple "ok" logs. Re-enable for debugging  | Error-only logging (current), success logging on every request (was too noisy)                  |
| A holdout use case changed to return `Result` type                      | All use cases should return `Result` for consistent error handling; this one was the last holdout     | Keep raw return (inconsistent with other use cases)                                             |

## Gotchas & Warnings

- Unauthenticated endpoints (e.g. public share/preview handlers) must NOT get `requireAuth()`. Use the `withErrorHandling` wrapper only.
- Endpoints that allow an optional session should keep the session lookup and return `null` (not throw) when unauthenticated.
- When a use case's return type changes from `Promise<Entity>` to `Promise<Result<Entity, SomeError>>`, update *all* callers (server function + API router).
- The wrapper adds a `code` field to error responses (`{ message, code }`). This is additive and backward-compatible — the frontend only accesses `.message`.
- Auth form components may still surface raw auth-library errors. These need an allowlist approach handled separately.
- Use cases in a shared `packages/application/` package use inline `console.error(JSON.stringify(...))` instead of `logError()` because they can't depend on app-specific imports.

## Dependencies

None — all utilities are project-internal. No new packages introduced.

## Testing

```bash
# Type-check web app
bunx tsc --noEmit -p apps/web/tsconfig.json

# Type-check API
bunx tsc --noEmit -p apps/api/tsconfig.json

# Manual testing
# 1. Trigger a DB error — verify user sees "Something went wrong", not SQL
# 2. Check the platform's log stream — verify structured JSON with fn, error, duration_ms
# 3. Sign out from the error page — verify sign-out button works on ErrorFallback
```

## Related

- Observability architecture doc — full error flow diagram and platform log patterns
- Client-side error tracking via an analytics platform (complementary to server-side logging)
