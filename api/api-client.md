# oRPC API Client

_Isomorphic oRPC client for the web app with JsonifiedClient, createIsomorphicFn, cookie forwarding, and TanStack Query utils._

## Purpose

`apps/web/src/lib/api-client.ts` exports the typed `api` and `orpc` singletons used by the web app to call the API worker. The client is **isomorphic** — the same `api`/`orpc` import works in browser code (hooks, components, mutations) AND in SSR contexts (route loaders, server functions). TanStack Start's Vite plugin tree-shakes the server branch out of the browser bundle.

Mobile uses the same oRPC client over public HTTPS.

## The file

```typescript
import { createORPCClient } from "@orpc/client";
import type { ContractRouterClient } from "@orpc/contract";
import { OpenAPILink } from "@orpc/openapi-client/fetch";
import type { JsonifiedClient } from "@orpc/openapi-client";
import { createTanstackQueryUtils } from "@orpc/tanstack-query";
import { createIsomorphicFn } from "@tanstack/react-start";
import { getRequestHeaders, getRequestUrl } from "@tanstack/react-start/server";
import { contract, type Contract } from "@app/api-contract";
import { apiFetch } from "@/lib/api-fetch";

type Client = JsonifiedClient<ContractRouterClient<Contract>>;

const getLink = createIsomorphicFn()
  .client(() =>
    new OpenAPILink(contract, {
      url: `${window.location.origin}/api/v1`,
      fetch: (req, init) => globalThis.fetch(req, { ...init, credentials: "include" }),
    }),
  )
  .server(() =>
    new OpenAPILink(contract, {
      url: () => `${getRequestUrl().origin}/api/v1`,
      fetch: (input, init) => {
        const req = input instanceof Request ? input : new Request(String(input), init);
        const headers = new Headers(req.headers);
        const cookie = getRequestHeaders().get("cookie") ?? "";
        if (cookie) headers.set("cookie", cookie);
        return apiFetch(new Request(req, { headers }));
      },
    }),
  );

export const api: Client = createORPCClient(getLink());
export const orpc = createTanstackQueryUtils(api);
```

## How it works

### `createIsomorphicFn` — one client, two runtimes

`createIsomorphicFn()` from `@tanstack/react-start` returns a builder with a `.client()` and a `.server()` branch. TanStack Start's Vite plugin statically rewrites the call site so the browser bundle only ships the `.client()` body and the server bundle only ships the `.server()` body. That is why static imports of `@/lib/api-fetch` and `@tanstack/react-start/server` at the top of this file do NOT leak `cloudflare:workers` or `node:stream` into the browser bundle — they are referenced only from inside the `.server()` block, which the plugin tree-shakes out of the client.

This pattern replaces an earlier design where the oRPC client was browser-only and SSR went through server functions. SSR now uses the same `api`/`orpc` import, with cookie forwarding handled in the server branch.

#### `.client()` branch

In the browser, the client uses `window.location.origin` to build the URL and wraps `globalThis.fetch` with `credentials: "include"`. The browser cookie jar attaches the auth session automatically because requests target the same origin (`/api/v1/*` on the web Worker, which proxies via Service Binding to the API worker).

#### `.server()` branch

On the server (Cloudflare Worker during SSR), the client:

1. Reads the inbound request URL via `getRequestUrl().origin` so the link resolves the same host the browser hit. The `url` is a function so it is evaluated **per request** inside the active request context.
2. Forwards the inbound `cookie` header via `getRequestHeaders()` so the auth session reaches the API worker.
3. Routes the request through `apiFetch`, which uses a Service Binding fetch helper to call the API directly via the Cloudflare Service Binding (zero-latency inter-worker, falling back to HTTP locally).

### `JsonifiedClient` — Date fields are strings on the wire

`JsonifiedClient<ContractRouterClient<Contract>>` wraps the typed client so consumer types reflect what JSON actually delivers. Contract `Date` fields become `string` at the call site; `bigint` becomes `string`; `undefined` is dropped. Treat date-shaped fields as strings everywhere consumers touch them, and only call `new Date(s)` at the leaf where a `Date` instance is genuinely needed (e.g. a date formatter).

This avoids the runtime mismatch between "the contract says `Date`" and "JSON gives you `string`" — TypeScript now matches reality. See [JSON wire vs Date — why dates are strings](#json-wire-vs-date--why-dates-are-strings) below for the full reasoning.

### `credentials: "include"` (client) vs cookie header forwarding (server)

In the browser, `credentials: "include"` is required for the auth cookie to attach. Removing it breaks authenticated requests silently (returns 401 without an obvious cause).

On the server there is no browser cookie jar. The server branch reads the inbound request's cookie header and copies it onto the outgoing request manually.

### `createTanstackQueryUtils(api)` — `orpc.x.y.queryOptions(...)`

`orpc` is the TanStack Query helper. It exposes `queryOptions` and `mutationOptions` builders for every contract route, fully typed:

- `orpc.groups.listGroups.queryOptions({ input: {} })` returns a ready-to-use `queryOptions` object
- `orpc.groups.createGroup.mutationOptions()` returns a ready-to-use `mutationOptions` object

The canonical pattern is to go through `orpc.*` rather than hand-rolling `queryFn: () => api.x.y(...)`. The helper builds the `queryKey` from the route path and input, handles the JSON unwrap, and ties into TanStack Query lifecycle correctly.

## SSR loader pattern

Route loaders run during SSR. Since the oRPC client is isomorphic, loaders can call `api` / `orpc` directly:

```typescript
import { api } from "@/lib/api-client";
import { groupKeys } from "@/hooks/use-groups";

export const Route = createFileRoute("/_authenticated/groups/")({
  loader: ({ context: { queryClient } }) =>
    queryClient.ensureQueryData({
      queryKey: groupKeys.list(),
      queryFn: () => api.groups.listGroups({}),
    }),
  component: GroupsPage,
});
```

The loader and the consumer hook share the **same `queryKey`**. The loader prefetches via `api` (server runtime, cookie forwarded automatically), and when the component renders, the hook reads the prefetched data from cache without firing a request. One data path, one cache slot.

## Hook authoring guide

Hooks live in `apps/web/src/hooks/use-*.ts`:

```typescript
import { orpc } from "@/lib/api-client";
import { useQuery, useMutation, useQueryClient } from "@tanstack/react-query";

export const groupKeys = {
  all: ["groups"] as const,
  list: () => [...groupKeys.all, "list"] as const,
  detail: (id: string) => [...groupKeys.all, "detail", id] as const,
};

export const useGroups = () =>
  useQuery({ ...orpc.groups.listGroups.queryOptions({ input: {} }), queryKey: groupKeys.list() });

export const useGroup = (id: string) =>
  useQuery({
    ...orpc.groups.getGroupDetail.queryOptions({ input: { groupId: id } }),
    queryKey: groupKeys.detail(id),
    enabled: !!id,
  });

export const useCreateGroup = () => {
  const qc = useQueryClient();
  return useMutation({
    ...orpc.groups.createGroup.mutationOptions(),
    onSuccess: () => qc.invalidateQueries({ queryKey: groupKeys.all }),
  });
};
```

Override `queryKey` only when the hook participates in an SSR-loader handshake (matching key). For pure-client queries the auto-generated key is fine.

## Importing API row types

Inferred row types live in `@app/api-contract` and are wrapped in `JsonifiedValue<>` so they match the wire (Date fields are `string`):

```typescript
import type {
  ExpenseRow,
  PaymentRow,
  SettlementRow,
  MemberBreakdown,
  GroupSummary,
  GroupDetail,
  GroupMember,
  Group,
} from "@app/api-contract";
```

**Web consumers must import row types from `@app/api-contract`, NOT from `@app/application/*`.** The application package returns Date instances; the wire delivers strings. The contract types are the right shape for components and hooks.

## JSON wire vs Date — why dates are strings

The server returns `{ createdAt: new Date() }`. `JSON.stringify` calls `Date.prototype.toJSON()` which produces an ISO string, so the wire payload is `{ "createdAt": "2026-05-06T00:00:00.000Z" }`. On the receiving side, `JSON.parse(text)` returns a string — there is no Date revival without an explicit reviver.

The contract uses `z.coerce.date()` for date fields, which would coerce a string to a Date if the schema were run at parse time. But `OpenAPILink` does NOT run output validation by default — in `@orpc/standard-server-fetch`, the response body is decoded with `await re.text()` followed by `JSON.parse(text)` with no reviver. The date stays a string.

`JsonifiedClient<ContractRouterClient<Contract>>` reflects this honestly: it walks the contract type and maps `Date` to `string`, `bigint` to `string`, and drops `undefined`. The type system now agrees with what `await api.groups.listGroups({})` actually returns at runtime.

### Migration history

Earlier in the project, server functions used TanStack Start's `devalue` serializer, which preserves Date instances on the wire. Components consumed Dates directly and `formatDate(date: Date)` accepted Date inputs.

When data fetching moved to oRPC + JSON, that contract broke. The fix was to:

1. Standardize all contract date fields on `z.coerce.date()`.
2. Re-introduce `JsonifiedClient` so call-site types are strings, not Date.
3. Export inferred row types from `@app/api-contract` wrapped in `JsonifiedValue<>` for consumer components.
4. Flip `formatDate(date: Date)` to `formatDate(date: string)` and parse internally.
5. Update local component types whose source is the API from `Date | null` to `string | null`.

### When would real Dates be on the wire?

If a structured serializer like SuperJSON were used, the wire could carry typed values that re-hydrate into Date, BigInt, Map, etc. oRPC's `RPCLink` supports custom serializers that do this. The choice here is deliberate: `OpenAPILink` keeps the API REST/OpenAPI-style — curlable, language-agnostic, mobile-friendly. The cost is that JSON-native types win, so consumers pay one `new Date(s)` at the leaf where a Date instance is needed.

## Key files

| Path                                     | Role                                              |
| ---------------------------------------- | ------------------------------------------------- |
| `apps/web/src/lib/api-client.ts`         | The isomorphic client (`api` + `orpc` exports)    |
| `apps/web/src/lib/api-fetch.ts`          | `apiFetch` via Cloudflare Service Binding         |
| `apps/web/src/hooks/use-*.ts`            | Hooks built on `orpc.*.queryOptions`              |
| `apps/web/src/server-functions/*.ts`     | Theme/cookie work (NOT the SSR data path for API) |
| `packages/api-contract/src/index.ts`     | Combined oRPC contract + inferred row types       |
| `apps/web/src/lib/format.ts`             | `formatDate(date: string)` accepts ISO strings    |

## Constraints

- Treat `Date`-shaped fields from the API as `string` (because `JsonifiedClient` says so). Cast with `new Date(...)` only at the leaf where a `Date` is needed.
- Build hook query/mutation specs through `orpc.x.y.queryOptions/mutationOptions` rather than hand-rolling `queryFn`. Override `queryKey` only when matching an SSR loader.
- Web consumers import API row types from `@app/api-contract`, never from `@app/application/*`.
- Don't introduce a dynamic `import("@tanstack/react-start/server")` from a module that runs in both client and server; rely on `createIsomorphicFn` instead so the Vite plugin can statically tree-shake.

## Gotchas

- `JsonifiedClient` removes `undefined` from optional fields. If a contract field is `T | undefined`, the client surface shows `T | null` or absence depending on serialization.
- The server branch's `url` is a **function** (not a string) so it evaluates inside the active request context; if you replace it with a constant, SSR loses access to the inbound origin.
- Server functions still exist for theme cookies and similar non-API work (`get-theme.ts`, `set-theme.ts`). They are NOT the SSR data path for API resources anymore.

## Related

- [ADR 0002: oRPC + Hono + Cloudflare](./decisions/adr-0002-orpc-hono-cloudflare.md) — Why this API layer exists
- [API Contract Pattern](./api-contract-pattern.md) — Where the contract lives and how it is consumed
- [Hono + oRPC stack](./hono.md) — Full stack overview
