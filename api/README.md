# API Knowledge

Patterns for a type-safe API layer: **Hono + oRPC on Cloudflare Workers**, contract-first routing, an isomorphic type-safe client, and the two auth topologies.

The default is a two-Worker architecture — a Hono API worker and a TanStack Start web worker — where the web app proxies `/api/auth/*` and `/api/v1/*` to the API worker (Service Bindings in production, plain `fetch()` in local dev), so requests stay same-origin.

When one identity has to be shared across products in separate repos, that proxy is not available and the service is called cross-origin instead. That topology has its own rules — see [centralized-auth-service.md](./centralized-auth-service.md).

## Contents

| File | Summary |
| ---- | ------- |
| [hono.md](./hono.md) | Full Hono + oRPC stack: contract, router, client, proxy. |
| [api-contract-pattern.md](./api-contract-pattern.md) | Contract in server app; extracting to a package. |
| [api-client.md](./api-client.md) | Isomorphic oRPC client, JsonifiedClient, cookie forwarding. |
| [centralized-auth-service.md](./centralized-auth-service.md) | Headless auth service for N products: cross-origin triad, KV session cache. |
| [decisions/adr-0002-orpc-hono-cloudflare.md](./decisions/adr-0002-orpc-hono-cloudflare.md) | Why oRPC + Hono replaced server functions. |
| [gotchas/better-auth-cross-origin.md](./gotchas/better-auth-cross-origin.md) | Immutable request headers on Cloudflare Workers. |

## Reading order

1. Start with **[hono.md](./hono.md)** for the whole picture.
2. Read **[api-contract-pattern.md](./api-contract-pattern.md)** to understand where the contract lives and why.
3. Read **[api-client.md](./api-client.md)** for the client-side details (SSR, dates on the wire, hooks).
4. Read **[centralized-auth-service.md](./centralized-auth-service.md)** only if identity spans several products — it replaces the proxy with a cross-origin boundary.
5. **[ADR 0002](./decisions/adr-0002-orpc-hono-cloudflare.md)** records the decision and its performance validation.
6. Consult **[gotchas](./gotchas/)** when auth misbehaves on Workers.
