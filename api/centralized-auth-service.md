# Centralized auth service (headless, cross-origin)

_One auth service owning identity for N independent products, instead of each product embedding its own. The topology, the three settings that must agree, and where session state lives._

> **Context:** Hono + Better Auth on Cloudflare Workers, Postgres for durable state, Workers KV as a session cache. The consuming products live in **other repositories** and call the service over HTTPS.

## When this instead of the auth proxy

[hono.md](./hono.md) documents the default topology: a web Worker and an API Worker in the same monorepo, where the web app proxies `/api/auth/*` to the API over a Service Binding. Requests are same-origin, so cookies "just work" and CORS is nearly an afterthought.

That falls apart the moment a **second** product needs the same users. Choose between:

| | Auth proxy (default) | Centralized auth service |
| --- | --- | --- |
| Products sharing identity | One | Many, in separate repos |
| Request origin | Same-origin via Service Binding | Cross-origin, always |
| Cookie setup | Default `sameSite: "lax"` works | `sameSite: "none"` + `secure`, mandatory |
| Cost of a new product | New auth deployment | One entry in an allowlist |
| Blast radius of an outage | One product | Every product |

The centralized service trades a shared point of failure for a single source of truth about who a user is. Take it when identity genuinely spans products; the proxy is simpler for everything else.

## The topology

```
consumer-web-a.example.com ─┐
consumer-web-b.example.com ─┼──── HTTPS (cross-origin, credentialed) ──► auth.example.com
mobile app ─────────────────┘                                             (Hono Worker)
                                                                              │
                                                              ┌───────────────┼───────────────┐
                                                              ▼                               ▼
                                                    Postgres (durable)              Workers KV (cache)
                                                    users, sessions,                hot session lookups
                                                    accounts, verification
```

There is no proxy and no Service Binding, because the consumers are not in this monorepo and are not even necessarily on Cloudflare.

## The three settings that must agree

This is the whole gotcha. Cross-origin credentialed auth needs three independent knobs pointing at the same truth, and a mismatch in any one of them fails in a way that looks nothing like an auth bug.

**1. CORS allowlist** — driven by config, never hardcoded:

```ts
app.use(
  "*",
  cors({
    origin: env.CORS_ORIGIN, // comma-separated list, parsed at startup
    allowMethods: ["GET", "POST", "PUT", "DELETE", "OPTIONS", "PATCH"],
    allowHeaders: ["Content-Type", "Authorization", "Cookie"],
    credentials: true,
    maxAge: 86400,
  }),
);
```

`credentials: true` makes a wildcard origin illegal — browsers reject `Access-Control-Allow-Origin: *` on credentialed requests. Every consumer must be listed explicitly. That is a feature: the allowlist becomes the registry of who is allowed to authenticate.

**2. Better Auth `trustedOrigins`** — the same list, or a caller that clears CORS gets rejected one layer deeper with a much worse error message:

```ts
export const auth = betterAuth({
  ...baseConfig,
  trustedOrigins: [...env.CORS_ORIGIN],
});
```

**3. Cookie attributes** — a session cookie without these never survives the hop to another domain:

```ts
advanced: {
  defaultCookieAttributes: {
    sameSite: "none",
    secure: true,
    httpOnly: true,
  },
},
```

Derive all three from **one** environment variable. Onboarding a product then means appending its origin to `CORS_ORIGIN` and redeploying — one change, not three files.

### What consumers must do

```ts
export const authClient = createAuthClient({
  baseURL: "https://auth.example.com",
  fetchOptions: {
    credentials: "include", // ← without this the cookie is silently omitted
  },
});
```

Omitting `credentials: "include"` is the most common integration failure. Nothing errors; every call simply looks unauthenticated.

### The third-party cookie caveat

`sameSite: "none"` is a *cross-site* cookie, and browsers are progressively restricting those. If the consumer and the auth service sit on unrelated registrable domains, the session can be dropped by default.

**Put both under one parent domain** — `app.example.com` and `auth.example.com`. The cookie is then same-site (different subdomain, same site) and survives regardless of third-party cookie policy. Design the DNS for this before writing code; retrofitting it means reissuing every session.

## Session storage: durable vs. cached

Sessions live in Postgres. Better Auth's `secondaryStorage` puts a read cache in front of it, which on Workers means KV:

```ts
export const auth = betterAuth({
  ...baseConfig,
  secondaryStorage: {
    get: (key) => sessionStore.get(key),
    set: (key, value, ttl) => sessionStore.set(key, value, { ttlSeconds: ttl }),
    delete: (key) => sessionStore.delete(key),
  },
});
```

`secondaryStorage` hands you already-serialized strings, so a typed JSON wrapper should be instantiated as `<string>` — it round-trips correctly, it just costs two quote characters.

A thin namespaced wrapper keeps the binding out of the auth config and makes the store reusable:

```ts
export function createKvStore<T>(namespace: KVNamespace, options: KvStoreOptions): KvStore<T> {
  const { prefix, defaultTtlSeconds } = options;
  const scoped = (key: string) => `${prefix}:${key}`;

  return {
    async get(key) {
      const raw = await namespace.get(scoped(key), "text");
      if (raw === null) return null;
      try {
        return JSON.parse(raw) as T;
      } catch {
        return null; // a poisoned key degrades to a miss, not a 500
      }
    },
    async set(key, value, setOptions) {
      const ttl = setOptions?.ttlSeconds ?? defaultTtlSeconds;
      await namespace.put(
        scoped(key),
        JSON.stringify(value),
        ttl === undefined ? undefined : { expirationTtl: Math.max(60, ttl) },
      );
    },
    async delete(key) {
      await namespace.delete(scoped(key));
    },
  };
}
```

Two boundaries worth internalizing:

- **KV is eventually consistent.** Perfect for a cache whose source of truth is the database. Wrong for anything that must be correct on the first read after a write — login-attempt counters, quotas, rate limits. Those need Durable Objects.
- **`expirationTtl` has a 60-second floor.** Clamp rather than letting Cloudflare reject the write.

The KV namespace is a binding, so it belongs in `wrangler.jsonc`, not in `.env`:

```jsonc
"kv_namespaces": [{ "binding": "AUTH_KV", "id": "REPLACE_ME" }]
```

`wrangler dev` simulates the namespace and ignores the id, so a placeholder is invisible locally and only fails at `wrangler deploy`. Ship it with an obviously-fake id and document `wrangler kv namespace create AUTH_KV` — a *loud* failure at deploy time beats a subtle one in production.

## Repo shape

A centralized auth service is a **backend-only** monorepo. The layering is unchanged (`domain <- application <- infra-*`, see [architecture/](../architecture/README.md)), but nothing renders UI:

```
apps/
  server-hono/       the auth API (the single composition root)
  documentation/

packages/
  domain/            schemas, types, repository interfaces
  application/       auth use cases
  infra-db/          Drizzle + Postgres
  infra-auth/        Better Auth config that is true regardless of deployment
  infra-cloudflare/  typed Worker bindings + the KV store
  infra-env/         Zod env validation
  i18n/              localized copy — no React
  config/
```

Two consequences that are easy to get wrong:

- **`infra-auth` vs. the app.** Keep environment-independent config in the package (database adapter, password hashing, cookie attributes) and environment-dependent wiring in the app (`trustedOrigins`, the KV binding, plugins). The app is the only layer allowed to know about both.
- **i18n has no React.** A headless service still renders localized strings — verification emails, OTP messages, push notifications, API error messages. Use `use-intl`'s non-React core (`createTranslator`) and drop the provider, the hooks, and the React peer dependency. Resolve the locale from a stored user preference first, falling back to `Accept-Language`.

## Prefix your tables

If the auth database is ever shared with another service, prefix every table and scope `tablesFilter` to that prefix:

```ts
export const createTable = pgTableCreator((name) => `${AUTH_TABLE_PREFIX}_${name}`);
```

```ts
tablesFilter: [`${AUTH_TABLE_PREFIX}_*`],
```

Without the filter, `drizzle-kit push` sees the other service's tables as "not in my schema" and offers to drop them. Decide the prefix before the first push; changing it later is a migration.

## Takeaways

- Centralize identity only when it genuinely spans products — the shared blast radius is real.
- **CORS allowlist, `trustedOrigins`, and cookie `sameSite` must be derived from one variable.** Three sources of truth guarantee a mismatch.
- Consumers must send `credentials: "include"`; forgetting it fails silently.
- Put the auth service and its consumers under one parent domain, so cookies stay same-site.
- KV caches sessions; Postgres owns them. Never put counters or rate limits in KV.
- A placeholder binding id that fails loudly at deploy beats one that fails subtly in production.
