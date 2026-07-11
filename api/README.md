# API Knowledge

Patterns for a type-safe API layer: **Hono + oRPC on Cloudflare Workers**, contract-first routing, an isomorphic type-safe client, and a same-origin auth proxy.

The stack is a two-Worker architecture — a Hono API worker and a TanStack Start web worker — where the web app proxies `/api/auth/*` and `/api/v1/*` to the API worker (Service Bindings in production, plain `fetch()` in local dev).

## Contents

| File | Summary |
| ---- | ------- |
| [hono.md](./hono.md) | Full Hono + oRPC stack: contract, router, client, proxy. |
| [api-contract-pattern.md](./api-contract-pattern.md) | Contract in server app; extracting to a package. |
| [api-client.md](./api-client.md) | Isomorphic oRPC client, JsonifiedClient, cookie forwarding. |
| [decisions/adr-0002-orpc-hono-cloudflare.md](./decisions/adr-0002-orpc-hono-cloudflare.md) | Why oRPC + Hono replaced server functions. |
| [gotchas/better-auth-cross-origin.md](./gotchas/better-auth-cross-origin.md) | Immutable request headers on Cloudflare Workers. |

## Reading order

1. Start with **[hono.md](./hono.md)** for the whole picture.
2. Read **[api-contract-pattern.md](./api-contract-pattern.md)** to understand where the contract lives and why.
3. Read **[api-client.md](./api-client.md)** for the client-side details (SSR, dates on the wire, hooks).
4. **[ADR 0002](./decisions/adr-0002-orpc-hono-cloudflare.md)** records the decision and its performance validation.
5. Consult **[gotchas](./gotchas/)** when auth misbehaves on Workers.
