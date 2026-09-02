# Stack: Service-only Hono (backend with no client)

_A backend service deployed as a single Cloudflare Worker (Hono + oRPC), with no web or
mobile app in the monorepo. The consumers live in **other repos** and call it over HTTPS.
The canonical case is a centralized auth service._

## When to use it

When a capability has to be shared by several products and cannot live inside
any one of them: identity, billing, notifications. If only one product consumes the
service, do not split it out — use [fullstack-hono-orpc](./fullstack-hono-orpc.md), which
also gives you same-origin cookies for free.

The structural difference from the other stacks is that **there is no proxy and no Service
Binding**: every consumer arrives cross-origin. That turns auth, cookies and CORS from a
"detail" into a "design constraint" — see [centralized auth service](../api/centralized-auth-service.md).

## Reading list (in order)

**1 · Foundations**
- [DDD + hexagonal architecture](../architecture/README.md) — the dependency rule
- [Domain modeling strategy](../architecture/domain-modeling-strategy.md) — how much ceremony to apply
- [Repository pattern](../architecture/repository-pattern.md) — interface in domain, implementation in infra
- [Monorepo structure](../monorepos/monorepo-structure.md) — Turborepo + Bun workspaces
- [The `infra-*` convention](../packages/infrastructure-naming.md) and the [build strategy](../packages/shared-package-build-strategy.md)

**2 · API (Hono + oRPC)**
- [Hono + oRPC overview](../api/hono.md) — the server stack (ignore the web/proxy part)
- [The api-contract pattern](../api/api-contract-pattern.md) — where the contract lives
- [ADR: oRPC + Hono + Cloudflare](../api/decisions/adr-0002-orpc-hono-cloudflare.md) — why this layer

**3 · The cross-origin service (what is specific to this stack)**
- [Centralized auth service](../api/centralized-auth-service.md) — topology, the CORS/`trustedOrigins`/cookies triad, KV as the session cache
- [Gotcha: better-auth cross-origin](../api/gotchas/better-auth-cross-origin.md) — immutable headers on Workers
- [Wrangler and env config](../monorepos/wrangler-env-config.md) — `.env` as the single source of truth
- [Migrating a domain to Workers](../infra/custom-domain-migration.md) — put the service and its consumers under the same parent domain

**4 · Errors and observability**
- [Result types](../error-handling/result-types.md) → [response helpers](../error-handling/response-helpers.md) → [api-response-types](../error-handling/api-response-types.md) → [error handlers](../error-handling/error-handlers.md)
- [Observability](../architecture/observability.md) and [security hardening](../architecture/security-hardening.md)

**5 · CI/CD and testing**
- [CI/CD per project](../monorepos/ci-cd-pipelines.md), [PR checks](../monorepos/pr-checks.md), [release-please](../monorepos/release-automation.md)
- [Testing strategy](../monorepos/testing-strategy.md)

## Assembly notes (what is specific to this stack)

- **A single Worker.** The app *is* the backend, so the deploy workflow has a single
  `wrangler deploy`; there is no Pages step and no client build.
- **CORS is the real access boundary.** There is no proxy hiding it. The allowlist comes from
  `CORS_ORIGIN` and also feeds `trustedOrigins`; with `credentials: true` a wildcard is
  illegal. Onboarding a consumer = one more entry in that variable.
- **Cookies with `sameSite: "none"` + `secure`.** And put the service and its consumers under the
  same parent domain (`auth.example.com` / `app.example.com`) so they stay same-site
  despite third-party cookie policies.
- **No React in any package.** `i18n` serves transactional copy (emails, push, error
  messages) through `use-intl`'s non-React core; drop the provider, the hooks and React as a peer dep.
- **`application` may start empty.** A barrel with `export {}` marking where the use cases
  will go is legitimate scaffolding, not residue — document that it is intentional.
- **Prefix the tables** and scope `tablesFilter` to that prefix, so `drizzle-kit push`
  never proposes dropping tables belonging to another service sharing the database.
- **Bindings in `wrangler.jsonc`, config in `.env`.** KV, Durable Objects and queues are
  bindings; secrets and URLs are not. Never a `vars` block.

## Generating it

`bun run customize` in [monorepo-template](https://github.com/niway-dev/monorepo-template)
offers this stack as **Backend only — Hono + oRPC API**: it keeps `apps/server-hono` and
`packages/i18n`, deletes web, mobile, `web-ui` and `tokens`, and generates the single-Worker
deploy workflow.
