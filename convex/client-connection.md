# Convex client connection

_How a client (web or Expo) connects to a Convex deployment: the client, providers,
env vars, the generated API, and monorepo module resolution._

## The client

One `ConvexReactClient` per app, created from the deployment's **data URL**
(`.convex.cloud`):

```ts
// src/lib/convex.ts
import { ConvexReactClient } from "convex/react";
const url = process.env.EXPO_PUBLIC_CONVEX_URL; // web: import.meta.env.VITE_CONVEX_URL
if (!url) throw new Error("Missing CONVEX_URL — copy .env.example to .env");
export const convex = new ConvexReactClient(url, { unsavedChangesWarning: false });
```

## Two URLs — don't mix them

| Env var | Origin | Used by |
| --- | --- | --- |
| `EXPO_PUBLIC_CONVEX_URL` | `.convex.cloud` | The **data** connection (`ConvexReactClient`, queries/mutations, the reactive WebSocket) |
| `EXPO_PUBLIC_CONVEX_SITE_URL` | `.convex.site` | Better Auth's `/api/auth/*` routes (the `authClient` baseURL) |

(Web uses `VITE_`-prefixed equivalents.)

## Providers

**Data only** (no auth): wrap the app in `ConvexProvider`.

```tsx
import { ConvexProvider } from "convex/react";
<ConvexProvider client={convex}>{children}</ConvexProvider>
```

**Data + auth** (Better Auth in Convex — see [better-auth](./better-auth.md)): use
`ConvexBetterAuthProvider`.

```tsx
import { ConvexBetterAuthProvider, type AuthClient } from "@convex-dev/better-auth/react";
// Expo: the cast is required (expoClient() type friction, get-convex/better-auth#168)
<ConvexBetterAuthProvider client={convex} authClient={authClient as unknown as AuthClient}>
  {children}
</ConvexBetterAuthProvider>
```

The Expo `authClient`:

```ts
// src/lib/auth-client.ts
import { convexClient } from "@convex-dev/better-auth/client/plugins";
import { expoClient } from "@better-auth/expo/client";
import { createAuthClient } from "better-auth/react";
import Constants from "expo-constants";
import * as SecureStore from "expo-secure-store";

const scheme = Constants.expoConfig?.scheme as string;
export const authClient = createAuthClient({
  baseURL: process.env.EXPO_PUBLIC_CONVEX_SITE_URL, // the .convex.site origin
  plugins: [
    expoClient({ scheme, storagePrefix: scheme, storage: SecureStore }),
    convexClient(),
  ],
});
```

(Web's authClient uses only `convexClient()` — no `expoClient()`, no cast needed.)

## Querying — reactive by default

```tsx
import { useQuery } from "convex/react";
import { api } from "@scope/backend/convex/_generated/api";
const categories = useQuery(api.categories.list, {}); // undefined while loading, then live
```

Convex queries are **subscriptions** — the component re-renders when the data changes.
No TanStack Query needed for reactive server state (use it only for what's outside Convex,
e.g. an offline-persisted cache).

## Consuming the generated API in a monorepo

The `api` object comes from the backend package's generated file:

```ts
import { api } from "@scope/backend/convex/_generated/api";
```

- `_generated/api.js` + `_generated/api.d.ts` are produced by `convex dev` (codegen).
  **They do not exist until the deployment is created** — a fresh backend needs
  `convex dev` (or `dev:setup`) to run once before an app can import from it.
- For **Metro (Expo)** to resolve a workspace backend package from source, the app needs a
  monorepo-aware `metro.config.js` — see [Expo dev builds & Metro](../mobile/expo-dev-builds-and-metro.md#monorepo-metro-config).

## First-time deployment (auth-in-Convex example)

```bash
# 1. Create the deployment + run codegen (generates _generated/api)
bun --filter @scope/backend dev:setup   # convex dev --configure --until-success
# 2. Set the site URL (Better Auth reads process.env.SITE_URL)
cd packages/backend && npx convex env set SITE_URL https://<deployment>.convex.site
# 3. Point the app at it
cp apps/mobile/.env.example apps/mobile/.env  # fill CONVEX_URL (.cloud) + SITE_URL (.site)
```

## Sanity checks

```bash
curl -s <deployment>.convex.cloud/version              # HTTP 200 → deployment up
curl -s <deployment>.convex.site/api/auth/get-session  # 200 "null" → auth deployed, no session
```

## Related

- [Better Auth in Convex](./better-auth.md) — the server + the auth version constraints.
- [Expo dev builds & Metro](../mobile/expo-dev-builds-and-metro.md) — running it on device.
