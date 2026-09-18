# Infisical + varlock: secrets and env playbook

> Adoption status: first applied to `invisible-assistant`
> (`feat/infisical-varlock-migration`, 2026-09-17). Sections marked **proven**
> were executed and verified there; the rest is the designed path, to be
> hardened as that migration (and the next adoptions) complete. Keep this doc
> updated from each project's migration log.

One source of truth for secret **values** (Infisical), one committed schema per
app declaring what env **exists** (varlock's `.env.schema`), and the same flow
on a laptop, a fresh laptop, GitHub Actions, and Cloudflare Workers.

**The golden rule:** the repo never contains secrets — not encrypted, not in
the clear. What travels with git is the schema. What travels outside git is a
single "secret zero" (one machine-identity credential in a gitignored
`.env.local`), or nothing at all in CI thanks to OIDC.

## Mental model

| Piece | What it does | Where it lives |
| --- | --- | --- |
| Infisical | Stores secret values, per project and per environment (`dev`, `staging`, `prod`) | Cloud (app.infisical.com) |
| `.env.schema` | Declares which variables exist, their type, sensitivity, and resolution source. Never holds sensitive values | Committed in the repo |
| varlock CLI | Reads the schema, resolves values (Infisical, local `.env.*`, literals), validates, injects into the process | Project dependency (+ optional global binary) |
| Machine identity | Infisical credential for software, not people. The "secret zero" | Host env var / OIDC in CI |

## Project setup in Infisical (once per product)

1. Create the project (one per **product**, not one global). Default envs
   `dev` / `staging` / `prod` are fine.
2. Load secrets per environment. Import an existing `.env` from the UI, or:
   ```bash
   infisical login
   infisical init
   infisical secrets set KEY=value --env=dev --path=/server
   ```
3. Create at least two machine identities (Access Control → Machine
   Identities):
   - `local-dev` — Universal Auth, **read-only on `dev`**. Store the
     client_id/client_secret pair in the password manager (shown once).
   - `github-actions` — OIDC auth, read-only on `staging` + `prod`.
     - Discovery URL / Issuer: `https://token.actions.githubusercontent.com`
     - Audience: `https://github.com/<org-or-user>`
     - Subject: `repo:<owner>/<repo>:*` (restrict tighter per-workflow when
       practical, e.g. `:environment:production`).
4. Note the **Project ID** (Project Settings) and the OIDC **Identity ID** —
   both are identifiers, not secrets, and get committed inside the schema.

### Organize by secret path (consumer, not app)

Group secrets by which *consumer* needs them — several apps often share the
same values. The layout that worked for a monorepo with a deployed API, a web
app, and a desktop release pipeline:

| Path | Contents | Envs |
| --- | --- | --- |
| `/server` | `DATABASE_URL`, `BETTER_AUTH_SECRET` | `dev` + `prod` |
| `/deploy` | `CLOUDFLARE_API_TOKEN`, `CLOUDFLARE_ACCOUNT_ID`, direct DB URL for migrations | `prod` |
| `/desktop-release` | Code-signing + notarization + artifact-store credentials | `prod` (+ a `dev` copy — see below) |
| `/ci` | `TEST_DATABASE_URL` and other CI-only secrets | `staging` |

**Least-privilege wrinkle (proven decision):** if a *local* machine sometimes
needs prod-shaped secrets (e.g. a locally-run signed desktop build), do NOT
give `local-dev` prod access. Copy those specific secrets into `dev` under the
same path instead. Local resolves `dev`, CI resolves `prod`, the identity
roles stay clean.

### Environment map

| `APP_ENV` (varlock) | Infisical env | Used by |
| --- | --- | --- |
| `development` | `dev` | laptops |
| `preview` | `staging` | PR/test CI jobs |
| `production` | `prod` | deploy + release workflows |

## varlock setup in the repo

**Pin exact versions** — varlock releases frequently. In a bun monorepo, pin
in the root `catalog` (versions as of 2026-09-17, **proven** to install and
run cleanly together):

```jsonc
"catalog": {
  "varlock": "1.19.0",
  "@varlock/infisical-plugin": "2.1.1",
  "@varlock/cloudflare-integration": "1.5.2"
}
```

- Root `devDependencies`: `varlock` + `@varlock/infisical-plugin`.
- `@varlock/cloudflare-integration` goes per-app, only where something deploys
  a Cloudflare Worker.
- Verify with `bunx varlock --version`.
- VS Code: recommend the `env-spec` extension in `.vscode/extensions.json`.
- Optional global binary: `brew install dmno-dev/tap/varlock`.

### Monorepo schema structure

One root `.env.schema` with the shared pieces; each app has its own schema
that `@import`s the root.

Root schema — plugin init, secret zero, current-env switch:

```bash
# @plugin(@varlock/infisical-plugin)
# @initInfisical(id=dev,  projectId=PROJECT_ID, environment=dev,  clientId=$INFISICAL_CLIENT_ID, clientSecret=$INFISICAL_CLIENT_SECRET, cacheTtl="1h")
# @initInfisical(id=prod, projectId=PROJECT_ID, environment=prod, identityId=IDENTITY_ID)
# @initInfisical(id=ci,   projectId=PROJECT_ID, environment=staging, identityId=IDENTITY_ID)
# @defaultSensitive=false @defaultRequired=infer @currentEnv=$APP_ENV
# ---

# @type=enum(development, preview, production, test)
APP_ENV=development

# --- secret zero: only set in .env.local; CI uses OIDC ---
# @type=string @optional
INFISICAL_CLIENT_ID=
# @type=string @sensitive @internal @optional
INFISICAL_CLIENT_SECRET=
```

App schema — its own variables, typed, with per-env resolution:

```bash
# @import(../../.env.schema)
# @generateTsTypes(path='env.d.ts')
# ---

# @type=url @sensitive
DATABASE_URL=forEnv(production, infisical(prod, path="/server"), infisical(dev, path="/server"))
# @sensitive
BETTER_AUTH_SECRET=forEnv(production, infisical(prod, path="/server"), infisical(dev, path="/server"))
# non-sensitive config stays as committed literals, NOT in Infisical
CORS_ORIGIN=http://localhost:3001
```

Notes:

- `@internal` on the client secret: varlock uses it to authenticate but never
  injects it into the app.
- `@sensitive` enables log redaction and leak detection.
- `@generateTsTypes` emits `env.d.ts` — add it to the app's tsconfig
  `include`, then `import { ENV } from 'varlock/env'` is typed and coerced.
- `infisical()` with no name uses the variable's own name as the secret name.
- Non-sensitive per-env values (public URLs, flags) live in the schema or in
  committed `.env.production` files — Infisical only holds what is secret.

### Local files

| File | Committed | Contents |
| --- | --- | --- |
| `.env.schema` (root + per app) | ✅ | The schemas |
| `.env.production` etc. | ✅ | Non-sensitive per-env overrides |
| `.env.local` | ❌ (gitignored) | Only `INFISICAL_CLIENT_ID` / `INFISICAL_CLIENT_SECRET` |
| `.env`, `.dev.vars` | gone | Deleted at the end of migration |

**Gitignore check:** make sure the pattern that ignores `.env*` does not also
ignore `.env.schema` (add `!.env.schema` if needed).

## Daily flow

```bash
varlock load                 # validate + show resolved env (secrets masked)
varlock run -- bun dev       # inject resolved env into a process
varlock scan                 # find leaked secrets in the code
```

Wire `varlock scan --staged` (or the closest supported invocation) into the
pre-commit hook, *before* lint-staged.

**New machine:** clone → install varlock → create `.env.local` with the pair
from the password manager → `varlock load`. Done. No pull step, no encrypted
file to sync.

**Changing a secret:** change it in Infisical; every machine and pipeline sees
it on the next run (within `cacheTtl`). New secret → also add its line to the
schema, or varlock flags it as missing.

## GitHub Actions (OIDC — zero secrets in GitHub)

Two mechanisms, by job shape:

**Jobs that run varlock anyway** (build + deploy): give the job
`permissions: { id-token: write, contents: read }`, set `APP_ENV`, and let the
schema's OIDC instance resolve everything. Run `bunx varlock load` as an early
step to fail fast on schema/Infisical mismatches.

```yaml
permissions:
  id-token: write
  contents: read
steps:
  - run: bunx varlock load
    env: { APP_ENV: production }
  - run: bunx varlock-wrangler deploy
    env: { APP_ENV: production }
```

**Jobs that just need env vars** (test runners): the Infisical Secrets Action.

```yaml
- uses: Infisical/secrets-action@v1
  with:
    method: oidc
    identity-id: IDENTITY_ID
    project-slug: <project-slug>
    env-slug: staging
    secret-path: /ci
```

Even the Cloudflare deploy token moves to Infisical (`/deploy`) — declare it
`@sensitive @optional` in the deploying app's schema and wrangler reads it
from the injected environment. GitHub Secrets end up literally empty.

**Do not empty GitHub Secrets until each migrated workflow has one verified
green run on the new path.** Until then both paths work and rollback is a
`git revert`.

## Cloudflare Workers

`@varlock/cloudflare-integration` provides `varlock-wrangler` (a wrangler
wrapper) and a Vite plugin (`varlockCloudflareVitePlugin()` wraps
`@cloudflare/vite-plugin` — pass it the same options).

Sharp edges:

- `wrangler.jsonc` needs `"compatibility_flags": ["nodejs_compat"]`.
- Delete `.dev.vars` / `.dev.vars.*` — they conflict with the plugin.
- `varlock-wrangler deploy` **replaces all** worker vars and secrets with the
  schema's. Anything set manually via `wrangler secret put` disappears on the
  next deploy. KV/D1/R2 bindings are untouched.
- Keep the existing rule: never add a `vars` block to `wrangler.jsonc` —
  varlock now owns vars end to end.

## Migration order for an existing project

1. Pin + install varlock packages; write the ADR; start a migration log doc.
2. Populate Infisical (paths above); create both identities.
3. Root `.env.schema` + `.env.local` → first green `varlock load` (validates
   plugin auth before any app is touched).
4. Migrate apps one at a time: app schema → swap the env module → switch
   scripts to `varlock run --` / `varlock-wrangler` → delete the old `.env` /
   `.env.example` only after `varlock load` and a dev boot pass.
5. Guardrails: `varlock scan` pre-commit, editor extension.
6. Workflows: OIDC + `varlock load` fail-fast; verify one green run each.
7. Gated cleanup: empty GitHub Secrets, retire dotenvx, mark the ADR accepted.

### Traps found in the first migration (proven)

- **Hunt down every `.env` consumer before deleting the file.** The obvious
  ones are the app's own env modules — but root-level scripts can load the
  same file too (in `invisible-assistant`, the root `db:push` / `db:studio` /
  `db:generate` / `db:migrate` scripts ran `dotenvx run -f
  apps/server-hono/.env`). Grep the whole repo for the file path, not just the
  app folder.
- **Keep the old validation layer deprecated-in-place, not deleted.** Marking
  the zod env package deprecated (comment + package description) costs a few
  lines of dead code and makes the revert trivial while the migration proves
  itself.
- **varlock decorator syntax moves fast.** Treat the exact decorator/resolver
  spellings in any doc (this one included) as "verify against the installed
  version"; record what actually parsed in the project's migration log.

## Security habits

- Machine identities with minimal permissions; `local-dev` must never see
  `prod`.
- Losing a laptop = rotating one client secret in Infisical.
- The schema gives AI agents names, types and descriptions without exposing
  values — safe context by construction.
- Enable secret versioning + audit logs (free tier) for rollback of values.

## References

- Infisical plugin: <https://varlock.dev/plugins/infisical/>
- Cloudflare integration: <https://varlock.dev/integrations/cloudflare/>
- GitHub Action: <https://varlock.dev/integrations/github-action/>
- OIDC guide: <https://varlock.dev/guides/oidc/>
- Environments: <https://varlock.dev/guides/environments/>
- Monorepos: <https://varlock.dev/guides/monorepos/>
- Infisical machine identities: <https://infisical.com/docs/documentation/platform/identities/machine-identities>
- First adoption: `invisible-assistant` — spec
  `docs/superpowers/specs/2026-09-16-infisical-varlock-migration-design.md`,
  log `docs/infisical-varlock-migration.md`, ADR 0014.
