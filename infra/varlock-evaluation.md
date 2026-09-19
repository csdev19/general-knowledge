# varlock: evaluated, not the default

> Status: **evaluated and set aside** (2026-09-18). `invisible-assistant` runs
> it; `kaipu-record-monorepo` started adopting it and stopped partway, keeping
> [Infisical alone](./infisical-secrets.md). This doc exists so the decision is
> not re-litigated from scratch, and so the parts that _are_ worth knowing
> survive.

**Documentation boundary:** setup, consumer tags, Worker files, build/signing
delivery, and CI migration belong to the standalone
[Infisical playbook](./infisical-secrets.md). Its
[README and variable-inventory contract](./environment-inventory.md) applies
whether or not Varlock is used. The snippets below document the evaluated
Varlock integration; they are not required Infisical onboarding steps.

## What varlock adds on top of Infisical

[varlock](https://varlock.dev) sits between your code and a secret store. It
adds a committed `.env.schema` declaring which variables exist, their type and
sensitivity, and where each resolves from:

```bash
# @plugin(@varlock/infisical-plugin)
# @initInfisical(id=dev, projectId=..., environment=dev, clientId=$INFISICAL_CLIENT_ID, clientSecret=$INFISICAL_CLIENT_SECRET)
# @defaultSensitive=false @defaultRequired=infer @currentEnv=$APP_ENV
# ---
DATABASE_URL=forEnv(production, infisical(prod, "DATABASE_URL", "/database"), infisical(dev, "DATABASE_URL", "/database"))
CORS_ORIGIN=forEnv(production, "https://example.com", "http://localhost:3001")
```

Genuinely useful properties:

- **A committed declaration of what env exists** that can be validated and typed,
  unlike a purely illustrative `.env.example`; consumer coverage still needs checks.
- **Type validation and generated TS types** (`@generateTsTypes`).
- **`varlock scan`** — leak detection for secrets committed in plaintext.
- **Mixed resolution** — secrets from Infisical, non-secrets as literals, in one
  file.
- **Name mapping** — one stored value feeding several variable names, so
  duplicates in the store collapse without touching application code.

## Why it is not the default

Three reasons, in order of weight.

**1. The schema often already exists.** A project with a zod env module —
`serverEnvSchema.parse(env)` and friends — already has a committed, typed
declaration of every variable, with real consumers. varlock replaces a working
TypeScript schema with a bespoke DSL and leaves the original as deprecated dead
code. That is a downgrade dressed as an upgrade. Check for this **before**
adopting: if there is no validation layer, varlock's schema is worth much more.

**2. Cloudflare Workers need two different integrations.** Which one depends on
how the app runs:

| App shape                    | Integration                                                                       |
| ---------------------------- | --------------------------------------------------------------------------------- |
| `wrangler` directly, no Vite | `@varlock/cloudflare-integration/init`, which re-exports `ENV` from `varlock/env` |
| `@cloudflare/vite-plugin`    | `varlockCloudflareVitePlugin()`                                                   |

Dev injection happens over a named pipe. In a monorepo with both shapes this is
the most fragile part of the stack, and it is the cost that finally outweighed
the benefit.

**3. It moves fast, and the docs lag.** Two spellings from the official playbook
did not parse against varlock 1.19.0 / `@varlock/infisical-plugin` 2.1.1. Pin
exact versions and verify every decorator against the installed build.

There is also a softer signal worth taking seriously: if the person who will
maintain the secret layer finds it complicated, that is data. A secrets layer
nobody fully understands is one that bites at 3am.

## What was verified against 1.19.0

Recorded so a future adoption does not rediscover it:

- **`secretPath` is positional, not a named argument.** The real signature is
  `infisical(instanceId, secretName, secretPath)`. The documented
  `infisical(dev, path="/x")` does not parse.
- **The secret zero has dedicated types**: `@type=infisicalClientId` and
  `@type=infisicalClientSecret`, better than plain `@type=string`.
- **A comment line beginning with `@` is parsed as a decorator.** Prose must not
  start with `@`, or you get a misleading "decorator cannot be used twice"
  error.
- `@initInfisical` accepts `secretPath` as a per-instance default, `cacheTtl`,
  and `allowMissing` (which can itself be dynamic, e.g. `forEnv(development)`).
- `infisicalBulk()` with `@setValuesBulk` loads a whole path at once.

Versions that installed and ran cleanly together: `varlock` 1.19.0,
`@varlock/infisical-plugin` 2.1.1, `@varlock/cloudflare-integration` 1.5.2.

## When to reach for it anyway

- The project has **no existing env validation layer**, so the schema is net new
  value rather than a duplicate.
- You want **leak scanning** in a pre-commit hook and have nothing else doing it
  (otherwise: gitleaks, trufflehog).
- Several environments resolve from **different sources** — some Infisical, some
  local, some literal — and the `forEnv()` switch earns its complexity.
- No Cloudflare Workers, or only one Worker shape.

If none of those hold, [Infisical with the existing runtime schemas](./infisical-secrets.md)
can meet the project's delivery and validation needs with fewer integrations.
It does not automatically provide Varlock's scanning or generated types.

## References

- <https://varlock.dev>
- Infisical plugin: <https://varlock.dev/plugins/infisical/>
- Cloudflare integration: <https://varlock.dev/integrations/cloudflare/>
- OIDC guide: <https://varlock.dev/guides/oidc/>
