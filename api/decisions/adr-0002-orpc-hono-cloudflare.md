# ADR 0002: oRPC + Hono + Cloudflare Workers as Primary API Layer

_Replace TanStack Start server functions with a dedicated oRPC + Hono API worker, validated by HAR performance comparison._

## Status

| Field         | Value      |
| ------------- | ---------- |
| Status        | Accepted   |
| Date          | 2026-05-06 |
| Supersedes    | N/A        |
| Superseded by | N/A        |

## Context

The web app originally used TanStack Start server functions (`_serverFn`) as the primary data-fetching layer. Server functions were convenient — co-located with the frontend, no separate deployment, one less network hop — but had four structural problems that compounded as the app grew:

1. **Opaque URLs.** Server functions are identified by a SHA hash (`_serverFn/d6bd098a…`). Network traces, logs, and error reports are unreadable without mapping hashes back to source files.

2. **Web-only.** The mobile app cannot call server functions. A public HTTP API was inevitable once mobile development started; building it as an afterthought would mean duplicating all business logic.

3. **No formal contract.** Input and output types were shared ad hoc between the frontend and the server function handler. There was no machine-checkable boundary — a type mismatch would surface at runtime, not at build time.

4. **Fused deployment.** The API and the web app lived in the same Cloudflare Worker. Scaling, access control, and observability were all coupled.

## Decision

Introduce `apps/<app>-api/` — a dedicated Cloudflare Worker running **Hono** with **oRPC** for contract-first routing — as the primary API layer.

- All data fetching moves from server functions to typed oRPC routes with Zod-declared input/output schemas in `@app/api-contract`.
- The web app forwards `/api/v1/*` to the API worker via a Cloudflare Service Binding (`API_SERVICE`) in production, and via a plain `fetch()` against `VITE_SERVER_URL` in local dev (see the Cloudflare infra proxy helpers).
- Only two server functions remain: `get-theme.ts` and `set-theme.ts` — web-only cookie work that has no equivalent in the API.
- Application use cases stay in `packages/application/` and are called from oRPC route handlers, keeping the domain layer unchanged.

## Performance validation

After completing the migration, HAR network traces were captured from both architectures on the same page flows and compared.

### App data requests only (analytics excluded)

| Metric | Old (`_serverFn`) | New (oRPC/Hono) | Delta  |
| ------ | ----------------- | --------------- | ------ |
| Count  | 10                | 11              | —      |
| Mean   | 341 ms            | 358 ms          | +17 ms |
| Median | 384 ms            | 387 ms          | +3 ms  |
| Min    | 190 ms            | 178 ms          | −12 ms |
| Max    | 410 ms            | 437 ms          | +27 ms |
| Stdev  | 84 ms             | 92 ms           | ≈ same |

### Total session (all 25 requests, including analytics)

| Architecture | Total    | Delta            |
| ------------ | -------- | ---------------- |
| Old          | 9,015 ms | —                |
| New          | 7,358 ms | −1,657 ms (−18%) |

### Reading the numbers

The median delta of **+3 ms** is within measurement noise. The mean delta of **+17 ms** is explained entirely by the local dev proxy: in local dev, `apiFetch` falls back to a plain HTTP call against `VITE_SERVER_URL` (the API dev server), adding one round-trip. In production, the Cloudflare Service Binding routes the call in-memory, adding effectively 0 ms.

The old architecture also showed higher max latency (1,154 ms for an analytics session upload vs 658 ms new) and higher variance overall. The new architecture is more consistent.

**Conclusion from the data:** the refactor does not regress performance. The two architectures are equivalent at the median, and the new one has better production headroom due to the Service Binding eliminating the dev proxy overhead.

## Consequences

**Positive:**

- Human-readable API URLs (`/api/v1/groups`, `/api/v1/groups/:id/balances`) appear in network traces, logs, and error reports.
- Mobile app calls the same oRPC routes over public HTTPS — no duplication of business logic.
- Contract types in `@app/api-contract` are the machine-checked boundary between frontend and backend; mismatches surface at build time.
- The API worker can be deployed, scaled, and monitored independently of the web worker.
- Use cases in `packages/application/` are unchanged — the API layer is a thin router on top of the existing domain.

**Negative:**

- Local dev adds one HTTP round-trip (web dev server → API dev server). Both processes must be running for the app to work locally.
- The proxy pattern requires `cloudflare:workers` to be imported with `await import()` + try/catch in the web app to avoid crashes in local dev (top-level import of the Service Binding module fails outside Cloudflare).

**Neutral:**

- The two remaining server functions (`get-theme.ts`, `set-theme.ts`) are not part of the data API and are unaffected.

## Alternatives considered

### Keep server functions, add a separate public API

Run server functions for the web and build a parallel Hono API for mobile. Rejected: duplicates all business logic and produces two surfaces to maintain with no shared contract.

### GraphQL

Flexible querying, strong schema. Rejected: adds significant complexity (resolver layer, codegen, client library) for a product where the query patterns are well-known and stable. oRPC with explicit contracts gives the same type safety with far less ceremony.

### tRPC

Similar to oRPC but does not generate an OpenAPI spec. Rejected: the OpenAPI output from oRPC lets mobile and future third-party consumers use standard HTTP tooling without a tRPC client.

## References

- [oRPC API Client](../api-client.md) — Isomorphic client, JsonifiedClient, cookie forwarding
- [API Contract Pattern](../api-contract-pattern.md) — Where the contract lives and how it is consumed
- [Hono + oRPC stack](../hono.md) — Full stack overview
