# Stack: Fullstack Hono + oRPC

_Web (TanStack Start) + API (Hono + oRPC) deployed as two Cloudflare Workers,
connected by Service Bindings. End-to-end type safety. **The default stack.**_

## When to use it

Your default for fullstack web products: a typed API, same-origin auth cookies on
Cloudflare, a shared contract without publishing a package. Pick this one unless you need
realtime out of the box (→ [convex](./fullstack-convex.md)) or you prefer Eden Treaty
(→ [elysia](./fullstack-elysia-eden.md)).

## Reading list (in order)

**1 · Foundations**
- [DDD + hexagonal architecture](../architecture/README.md) — the dependency rule
- [Domain modeling strategy](../architecture/domain-modeling-strategy.md) — how much ceremony to apply
- [Repository pattern](../architecture/repository-pattern.md) — interface in domain, implementation in infra
- [Monorepo structure](../monorepos/monorepo-structure.md) — Turborepo + Bun workspaces
- [The `infra-*` convention](../packages/infrastructure-naming.md) and the [build strategy](../packages/shared-package-build-strategy.md)

**2 · API (Hono + oRPC)**
- [Hono + oRPC overview](../api/hono.md) — the full server stack
- [The api-contract pattern](../api/api-contract-pattern.md) — where the contract lives and how it is consumed
- [Isomorphic client](../api/api-client.md) — the same client on the server and in the browser
- [ADR: oRPC + Hono + Cloudflare](../api/decisions/adr-0002-orpc-hono-cloudflare.md) — why this layer
- [Gotcha: better-auth cross-origin](../api/gotchas/better-auth-cross-origin.md) — immutable headers on Workers

**3 · Web (TanStack Start)**
- [Data loading](../web/data-loading.md) — loaders + Query; server functions vs the isomorphic client
- [Server functions](../web/server-functions.md) — the boundary and its rules
- [Shared UI package](../web/web-ui-package.md) — shadcn/ui + OKLCH theming
- [Bundle splitting](../web/bundle-splitting.md) and [local preview](../web/local-preview.md)

**4 · Errors and observability**
- [Result types](../error-handling/result-types.md) → [response helpers](../error-handling/response-helpers.md) → [api-response-types](../error-handling/api-response-types.md) → [error handlers](../error-handling/error-handlers.md)
- [Observability](../architecture/observability.md) and [security hardening](../architecture/security-hardening.md)

**5 · CI/CD and testing**
- [CI/CD per project](../monorepos/ci-cd-pipelines.md), [PR checks](../monorepos/pr-checks.md), [release-please](../monorepos/release-automation.md)
- [Testing strategy](../monorepos/testing-strategy.md)

## Assembly notes (what is specific to this stack)

- **Same-origin proxy**: the web Worker proxies `/api/auth/*` and `/api/v1/*` to the API Worker.
  In production via Service Bindings (Worker-to-Worker, no public DNS); in local dev, `fetch()`.
  The `Set-Cookie` headers are rewritten to drop `domain=` and bind the cookie to the web domain.
- **The contract lives in the server app** (`apps/api/src/contract/`), not in a separate package.
  The web imports it by a cross-app relative path (the same pattern as Eden's `type { App }`).
  See [api-contract-pattern](../api/api-contract-pattern.md) if you ever want to extract it into a package.
- **`router.invalidate()`** after sign-in/up/out, to refresh the root's `beforeLoad`; otherwise
  the header shows stale auth state.
- **The web does NOT run Better Auth locally** — it proxies to the backend's auth. `infra-auth` is not
  a dependency of the web app.
- **The server's CORS is for mobile only** (`exp://`, `mobile://`); the web is same-origin through the proxy.
