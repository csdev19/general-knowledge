# Better Auth hosted inside Convex

_Auth that lives **in** the Convex deployment (via `@convex-dev/better-auth`), not on
a separate API server. Battle-tested on Expo SDK 57 / React 19 / RN 0.86._

## Two ways to wire auth with Convex

| Model | Where Better Auth runs | Convex's role | When |
| --- | --- | --- | --- |
| **Auth-in-Convex** (this doc) | Inside the Convex deployment, on its HTTP router | Data **and** auth from one deployment | Simplest for a Convex-first app; one backend to run |
| External auth server | A separate Node/Workers server (Hono/Elysia) | Data only; validates JWTs via `customJwt` in `auth.config.ts` | When auth is shared across non-Convex services |

This doc is the **auth-in-Convex** model. The `convex.config.ts` registers the
`betterAuth` component and the deployment serves `/api/auth/*` itself.

## The full setup (one Convex package)

```
convex/
  convex.config.ts   # app.use(betterAuth)  — registers the component
  auth.config.ts     # providers: [getAuthConfigProvider()]
  auth.ts            # createClient + createAuth(betterAuth({...})) + getCurrentUser
  http.ts            # authComponent.registerRoutes(http, createAuth)
  schema.ts, *.ts    # your data
```

```ts
// convex.config.ts
import betterAuth from "@convex-dev/better-auth/convex.config";
import { defineApp } from "convex/server";
const app = defineApp();
app.use(betterAuth);
export default app;
```

```ts
// auth.ts
import { createClient, type GenericCtx } from "@convex-dev/better-auth";
import { convex } from "@convex-dev/better-auth/plugins";
import { expo } from "@better-auth/expo";
import { betterAuth } from "better-auth";
import type { DataModel } from "./_generated/dataModel";
import { components } from "./_generated/api";
import { query } from "./_generated/server";
import authConfig from "./auth.config";

const siteUrl = process.env.SITE_URL!; // the .convex.site origin

export const authComponent = createClient<DataModel>(components.betterAuth);

function createAuth(ctx: GenericCtx<DataModel>) {
  return betterAuth({
    baseURL: siteUrl,
    trustedOrigins: [siteUrl, "native://", "exp://"], // see "Trusted origins" below
    database: authComponent.adapter(ctx),
    emailAndPassword: { enabled: true, requireEmailVerification: false },
    plugins: [convex({ authConfig }), expo()],
  });
}
export { createAuth };

export const getCurrentUser = query({
  args: {},
  handler: async (ctx) => authComponent.safeGetAuthUser(ctx),
});
```

```ts
// http.ts
import { httpRouter } from "convex/server";
import { authComponent, createAuth } from "./auth";
const http = httpRouter();
authComponent.registerRoutes(http, createAuth);
export default http;
```

**Deployment env:** set `SITE_URL` on the deployment to its `.convex.site` origin
(`npx convex env set SITE_URL https://<deployment>.convex.site`). Better Auth serves
its routes there; the client talks to it (see [client-connection](./client-connection.md)).

## ⚠️ The version constraints (do not drift)

The reactive layer is **version-fragile on React 19 / RN 0.86 / Expo SDK 57**. Pin these:

| Package | Version | Constraint |
| --- | --- | --- |
| `better-auth` | `1.6.23` | Stay `< 1.7.0` (1.7 has big OAuth/2FA/plugin breaking changes) |
| `@convex-dev/better-auth` | `0.12.5` | Peer-requires `better-auth >=1.6.11 <1.7.0` |
| `@better-auth/expo` | `1.6.23` | Lockstep with `better-auth`; runtime-requires **`expo-network`** |

**Rule:** upgrade `better-auth` and `@convex-dev/better-auth` **together**, only to a
combination whose peer range is satisfied.

## The SDK 57 bug this recipe fixes

On the **old** stack (`@convex-dev/better-auth@0.10.x` + `better-auth@1.4.x`) under
React 19.2 / RN 0.86 / Expo SDK 57, Better Auth's **reactive** `useSession()` never
resolved (`isPending` stuck `true`), which froze `useConvexAuth().isLoading` → the auth
gate hung on a black screen. The **imperative** `authClient.getSession()` worked; only
the nanostores-backed reactive hook was broken. Fixed by the version bump above.

_Ruled out during diagnosis (so you don't chase them again): app code (TanStack Query /
persist / routing), Convex itself (plain `ConvexProvider` + `useQuery` worked), React
Compiler, network, emulator clock/TLS._

## Three gotchas the version bump alone does NOT solve

1. **`expo-network` is a runtime dependency.** `@better-auth/expo@1.6.x` `require`s it;
   missing it throws `Cannot find module 'expo-network'` on start. Add `expo-network`
   (`~57.0.1`) to every Expo app. Type-check does **not** catch this — it's runtime-only.

2. **Dedupe transitive `@better-fetch/fetch` / `better-call`** via root `package.json`
   `overrides`: `"@better-fetch/fetch": "1.3.1"`, `"better-call": "1.3.7"`. Two copies
   otherwise resolve; the differing `BetterFetch` types break Better Auth client type
   unification (`not assignable to type AuthClient`). Symptom is compile-time only.

3. **Cast the Expo `authClient`** at the provider:
   `authClient={authClient as unknown as AuthClient}` (import `AuthClient` from
   `@convex-dev/better-auth/react`). The `expoClient()` plugin's inferred
   `useSession().data` shape is rejected at compile time though it works at runtime.
   Upstream: `get-convex/better-auth#168`. Web (no `expoClient()`) does not need this.

## Trusted origins — `exp://` is required for Expo Go dev

`trustedOrigins` must list every client origin:

- **Deploy path:** the `.convex.site` `siteUrl` (web / production).
- **App scheme:** `native://` (standalone / dev build).
- **`exp://`** — in **Expo Go**, requests carry an `exp://<host>:<port>` origin, NOT the
  app scheme. Better Auth returns `Invalid origin` (403) unless `exp://` is trusted, and
  `@better-auth/expo` only auto-trusts it when `NODE_ENV=development` — which a Convex
  deployment is **not**. A single `"exp://"` entry matches any `exp://host:port`.

> A `trustedOrigins` change only takes effect after the backend **redeploys**
> (`bun run dev:server`). If login still says `Invalid origin`, the deploy didn't run.

## Diagnosing an auth-loading hang

If a client sits forever on "loading" (`useConvexAuth().isLoading` never `false`):

1. **Is Convex connected?** Add `useQuery(api.healthCheck.get)`; if it returns `"OK"`,
   Convex + network are fine → the bug is in the auth layer, not Convex.
2. **Imperative vs reactive session.** Log `authClient.getSession()` (imperative) and
   `authClient.useSession()` (reactive). Imperative resolves but `useSession().isPending`
   stuck `true` ⇒ version incompatibility in the Better Auth stack, not app code.
3. **Reproduce in isolation.** A fresh minimal Expo app with only
   `ConvexBetterAuthProvider` + `authClient` tells you instantly whether it's app-specific.
4. Rule out React Compiler / network / clock before assuming a code bug.

## Related

- [Convex client connection](./client-connection.md) — the client side (`authClient`,
  `ConvexBetterAuthProvider`, env, monorepo Metro resolution).
- [Expo dev builds & Metro](../mobile/expo-dev-builds-and-metro.md) — running the app.
