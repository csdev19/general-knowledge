# Infisical: secrets and env playbook

> Adoption status: **accepted default.** Applied to `kaipu-record-monorepo`
> (2026-09-18), where it replaced dotenvx. `invisible-assistant` runs the
> varlock variant — see [varlock-evaluation.md](./varlock-evaluation.md) for
> why that extra layer is no longer the default.

One source of truth for secret **values** (Infisical), fetched at start-up by
the scripts that need them. No hand-maintained `.env` files anywhere.

**The golden rule:** the repo never contains secrets — not encrypted, not in the
clear. Where a `.env` exists it is **generated** and gitignored, and editing it
is always a mistake because the next start overwrites it.

## Mental model

| Piece | What it does | Where it lives |
| --- | --- | --- |
| Infisical | Stores secret values, per project and per environment | Cloud (app.infisical.com) |
| `.infisical.json` | Links the repo to a project. Holds a workspace id, not a credential | Committed |
| Infisical CLI | Fetches values and injects them into a process | A global tool, declared not installed — see [tool-doctor-pattern](../conventions/tool-doctor-pattern.md) |
| Your login / a machine identity | How the CLI authenticates | Local session, or OIDC in CI |

## Project setup (once per product)

1. Create the project — one per **product**, not one global.
2. Load secrets per environment. Import an existing `.env` from the UI, or
   `infisical secrets set KEY=value --env=dev --path=/server`.
3. `infisical init` in the repo root writes `.infisical.json`. Commit it: it
   contains only the workspace id, so a fresh clone points at the right project
   with no setup.

### Environment slugs are not environment names

The UI shows "Development"; the API wants `dev`. Passing the display name gives
a 404 that reads like a permissions problem:

```
Message: Environment with slug 'development' not found
```

Defaults are `dev`, `staging`, `prod`. Check Project Settings → Environments if
unsure. This cost real time in the Kaipu adoption — check it first.

### Organise by what a credential unlocks

Group by the **external system a credential opens**, not by the app that happens
to use it. Shared systems get their own path; what stays in an app path is
genuinely that app's own. Every credential then exists in exactly one place, so
rotation has exactly one target.

| Path | Contents |
| --- | --- |
| `/cloudflare` | Account id, R2 keys, deploy token |
| `/database` | `DATABASE_URL` (+ `TEST_DATABASE_URL`) |
| `/<app-name>` | That app's own: session secret, analytics key, mail key |

Root holds non-secret per-environment config.

**Resist the urge to file everything.** Folders of one or two variables are
filing for its own sake. In the Kaipu adoption five folders collapsed to four
once R2 was recognised as part of Cloudflare.

**What paths actually buy you.** The usual argument is per-path access control,
but that benefit is often *deferred*: the strong isolation comes from the
environment boundary — a laptop identity reads `dev`, CI reads `prod`. If every
CI workflow shares one GitHub Environment they produce an identical OIDC subject
and cannot be told apart by identity, however the paths are drawn. Adopt paths
for organisation and to keep a future split cheap, not on the belief that they
are isolating anything today.

### What does NOT belong in Infisical

Only what grants access. URLs, flags, labels, bucket names and account
identifiers are config, not credentials.

Two tests settle most arguments:

- **Is it already committed in plaintext?** If a workflow hardcodes it, storing
  it as a secret protects nothing. Kaipu had `R2_BUCKET: kaipu-bucket` written
  literally in a release workflow while also keeping it as a GitHub Secret.
- **Does keeping it in Infisical buy operational flexibility?** Usually not.
  Worker vars resolve at deploy time and desktop vars are inlined at build time,
  so changing a value in Infisical does nothing until the next deploy or
  rebuild — exactly what a committed literal already requires. Meanwhile the
  value stops appearing in pull-request diffs.

Roughly half of what a project treats as secret turns out to be config. In Kaipu
it was 15 of 32 variables.

## Wiring it up

Two mechanisms. Which one you need depends on how the consumer reads its env —
and this is the decision that shapes the whole migration.

### `infisical run` — anything reading `process.env`

```bash
infisical run --env=dev --recursive -- <command>
```

Covers repo-root scripts, `vite dev`, test runners, and **Vite-based builds
including Electron**: Vite's `loadEnv` reads `process.env` *after* the `.env`
files and lets it win, so prefixed variables (`VITE_*`, `MAIN_VITE_*`) are
picked up with no integration needed. Verified in
`vite/dist/node/chunks/config.js`:

```js
for (const [key, value] of Object.entries(parsed))   // .env files first
  if (prefixes.some(p => key.startsWith(p))) env[key] = value;
for (const key in process.env)                        // process.env second — wins
  if (prefixes.some(p => key.startsWith(p))) env[key] = process.env[key];
```

The discipline this imposes: names in Infisical must carry the **exact prefix**
the bundler expects. A mismatched prefix is ignored silently — a mute failure,
not a loud one. Verify by grepping the built bundle for an expected value, not
by a green exit code.

### A generated file — for Cloudflare Workers

A Worker reads **bindings**, not the parent process environment, so injecting
into `wrangler`'s process does not reach the Worker. Wrangler populates bindings
from a file, and modern versions read `.env` as well as `.dev.vars`
(`--env-file <path>` points at an arbitrary one).

```json
"env:pull": "infisical export --env=dev --path=/ --silent > .env && infisical export --env=dev --path=/database --silent >> .env",
"dev": "bun run env:pull && wrangler dev"
```

**`infisical export` has no `--recursive` flag — only `infisical run` does.** So
exporting across several paths means one invocation per path, concatenated. This
asymmetry is not documented anywhere obvious and is the most surprising thing in
the CLI.

Prefer writing the generated file somewhere it cannot be mistaken for
hand-maintained config. Writing it to the app's normal `.env` path works and is
the smallest change, but six months later nobody remembers which files are
generated.

## CI

Use the official action with OIDC, so no long-lived secret lives in GitHub:

```yaml
permissions: { id-token: write, contents: read }
steps:
  - uses: Infisical/secrets-action@v1
    with:
      method: oidc
      identity-id: <IDENTITY_ID>
      project-slug: <project-slug>
      env-slug: prod
      secret-path: /server
```

OIDC machine identity configuration:

| Field | Value |
| --- | --- |
| Issuer | `https://token.actions.githubusercontent.com` |
| Audience | `https://github.com/<org-or-user>` |
| Subject | `repo:<owner>/<repo>:environment:<name>` |

**On the subject claim:** when a job declares `environment: production`, GitHub
swaps the default `:ref:refs/...` subject for `:environment:production`. If your
deploy workflows already use a GitHub Environment, scope the identity to that
rather than the `:*` wildcard — strictly tighter at no cost. If several
workflows share one environment they are indistinguishable by subject; separate
GitHub Environments are what makes them separable.

**Do not empty GitHub Secrets until each migrated workflow has one verified
green run on the new path.** Until then both paths work and rollback is a
`git revert`.

### What stays in GitHub Secrets

A token that authenticates *to GitHub, from GitHub* — a release-please PAT, for
instance. Routing it through an external service adds a bootstrap dependency and
no protection, since GitHub already holds it encrypted. Accept the one non-empty
secret rather than chase a clean slate.

## Migration order for an existing project

1. Load secrets into Infisical and organise the paths. Commit `.infisical.json`.
2. Declare the CLI as a required tool
   ([tool-doctor-pattern](../conventions/tool-doctor-pattern.md)).
3. **Move existing `.env` files out of the tree** — to a gitignored scratch
   directory, not deleted. This is both a safety net and what makes the next
   step meaningful: with the old file still in place you cannot tell whether the
   app booted on Infisical's values or the leftover file's.
4. Swap the scripts one consumer at a time. Boot the app after each.
5. Migrate CI to OIDC; verify one green run per workflow.
6. Gated cleanup: empty the migrated GitHub Secrets, delete the `.env.example`
   files the migration made misleading, remove the scratch backups.

### Traps found in adoption

- **Hunt down every `.env` consumer before moving the file.** The obvious ones
  are the app's own env modules — but root-level scripts often load the same
  file. In Kaipu, seven repo-root `db:*` scripts ran
  `dotenvx run -f apps/server-hono/.env`. Grep the whole repo for the file
  **path**, not just the app folder.
- **Verify against an endpoint that touches the database**, not just a clean
  boot. A zod env schema failing to parse is loud; a subtly wrong
  `DATABASE_URL` is not.
- **Check for values the old file had that Infisical does not.** An optional
  variable with a production default is the dangerous shape: nothing fails, it
  quietly uses the production value in local dev. Kaipu nearly regressed an
  email-verification fix exactly this way.
- **Do not bundle credential rotation into the migration.** If something breaks
  you cannot tell which change caused it.

## Security habits

- Machine identities with minimal permissions; a laptop identity must never see
  `prod`.
- Keep the credential that grants deploy rights in `prod` only, so a
  laptop-scoped identity cannot deploy or destroy infrastructure.
- Losing a laptop means rotating one credential.
- Enable secret versioning + audit logs (free tier) for rollback of values.

## References

- CLI overview: <https://infisical.com/docs/cli/overview>
- GitHub Action: <https://infisical.com/docs/integrations/cicd/githubactions>
- Machine identities: <https://infisical.com/docs/documentation/platform/identities/machine-identities>
- The declaration layer that was evaluated and dropped:
  [varlock-evaluation.md](./varlock-evaluation.md)
