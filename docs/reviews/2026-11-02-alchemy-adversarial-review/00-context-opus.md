# Context package: should Kaipu move its Cloudflare deploys to Alchemy?

Prepared 2026-11-02 by the context reader. Neutral. Kaipu repo at `c01796bb` (main, 2026-11-01);
hub repo at `93134fe`. Paths: `K/` = `kaipu-record-monorepo/`, `H/` = `general-knowledge/`,
`T/` = `monorepo-template/` (all under `~/Documents/Workspace/Personal/`).

## 1. The question

Should Kaipu replace its current wrangler-based Worker deploys with Alchemy (alchemy.run), a
TypeScript infrastructure-as-code library whose newer line is built on Effect? "Replace" has three
readings, and the debate must say which one it argues: (a) **full IaC** — Alchemy declares and owns
every Cloudflare resource (Workers, routes/custom domains, R2 buckets, DNS, Access, state), and
`wrangler.jsonc` becomes generated or disappears; (b) **deploy-only** — Alchemy replaces the
`wrangler deploy` step for the three Workers while resources created by hand stay unmanaged;
(c) **hybrid** — Alchemy provisions resources (buckets, future KV/D1/queues, a future dev Worker)
while `wrangler deploy` keeps shipping code. Note up front: the premise "deploys via
`cloudflare/wrangler-action`" is only half true today — see F3.

## 2. Current pipeline, as-is

### Per app

| App | Worker name | Build | Deploy | Trigger / env selection | Secrets / vars source |
| --- | --- | --- | --- | --- | --- |
| `apps/server-hono` (API) | `kaipu-api` | `bun run build --filter='@kaipu/*'` (packages only; wrangler bundles `src/index.ts`) — `K/.github/workflows/release-api.yml:81-82` | `bash scripts/deploy-worker.sh apps/server-hono` → `wrangler@4.148.0 deploy --cwd … --secrets-file worker-secrets.json` — `release-api.yml:131-135`, `K/scripts/deploy-worker.sh:21,50` | tag `api-v*.*.*` or `workflow_dispatch` (`release-api.yml:7-11`); job `needs: verify` (`:65`); GitHub Environment `production` (`:67`) | GitHub `production` Environment secrets/vars → `jq` → `worker-secrets.json` (11 keys: DATABASE_URL, CORS_ORIGIN, BETTER_AUTH_SECRET, GOOGLE_CLIENT_ID/SECRET, BETTER_AUTH_URL, R2_ACCOUNT_ID, R2_ACCESS_KEY_ID, R2_SECRET_ACCESS_KEY, R2_BUCKET, RESEND_API_KEY) — `release-api.yml:91-125`; empty values dropped; file removed `if: always()` (`:137-139`) |
| `apps/web-hono` (landing + proxy) | `kaipu-web` | packages, then `bun run build` in app with VITE_SERVER_URL, DATABASE_URL, VITE_PUBLIC_DOWNLOAD_URL, VITE_POSTHOG_* — `K/.github/workflows/release-web.yml:81-94` | same script: `deploy-worker.sh apps/web-hono` — `release-web.yml:119-123` | tag `web-v*.*.*` or dispatch (`:7-11`); `needs: verify`; Environment `production` | `worker-secrets.json` with VITE_SERVER_URL, DATABASE_URL (`release-web.yml:105-115`; build `:84-94`) |
| `apps/web-hono` preview | `kaipu-web` (version only) | identical build (`K/.github/workflows/preview-web.yml:87-102`) | `cloudflare/wrangler-action@v3.14.1`, `command: versions upload --message …` (`:104-113`), wrangler `4.148.0` pre-installed with bun (`:79-80`) | manual `workflow_dispatch` with `ref` input (`:19-29`); Environment `preview` (`:49`); never deploys | Cloudflare token + account id; DATABASE_URL in `preview` env is a "deliberately unusable value" (`:91-97`, `K/apps/documentation/src/content/docs/deployment/ci-cd-pipelines.md:154`) |
| `apps/console` | `kaipu-console` | — | **no deploy workflow exists** (`grep console .github/workflows/*.yml` → only an unrelated comment) | — | — |
| `apps/kaipu-record` (desktop) | n/a | electron-builder | uploads to R2 bucket `kaipu-bucket` via S3 API with R2 keys (not a Worker) — `release-desktop.yml`; `deployment/ci-cd-pipelines.md:138-141` | tag `desktop-v*` | GitHub `production` |

- Release-candidate PRs dry-run both deploys with the same script: `K/.github/workflows/pr-checks.yml:304-312`.
- Tags are created by release-please (`K/.github/workflows/release-please.yml:1-12`, `K/release-please-config.json`: components desktop/web/api; console is not a component).
- Concurrency: release workflows `cancel-in-progress: true`; preview serialized, not cancelling (`preview-web.yml:31-35`).
- There is **no staging or dev environment**; no `env` blocks in any `wrangler.jsonc`. Only "environments" are GitHub Environments `production`, `preview`, `testing` (`pr-checks.yml:483`).
- Secrets: release workflows still read **GitHub Secrets**, not Infisical: "`prod` is untouched — the release workflows still read GitHub Secrets" (`K/apps/documentation/src/content/docs/deployment/secrets-layout.md:19`). Infisical has a single `dev` environment (`backlog/deploy-environments.md:36-37`).
- Wrangler versions: workspace devDependency `4.63.0` pinned directly in `apps/server-hono/package.json:37` and `apps/console/package.json:34` (not in the root Bun catalog — root catalog `K/package.json:12-31` has no wrangler); CI uses `4.148.0` (`deploy-worker.sh:21`, `preview-web.yml:80`). turbo.json has no deploy task. `apps/server-hono/package.json:10` still has a local `"deploy": "wrangler deploy"`.

### Resources declared in `wrangler.jsonc`

| File | Resources |
| --- | --- |
| `K/apps/server-hono/wrangler.jsonc:1-11` | name `kaipu-api`; compat date `2025-10-31`; flags `nodejs_compat`, `nodejs_compat_populate_process_env`; `preview_urls: true`; observability on. **No routes, no KV, D1, R2, DO, queues, vars, env blocks.** |
| `K/apps/web-hono/wrangler.jsonc:1-48` | name `kaipu-web`; compat `2025-10-04`; Workers Static Assets `dist/client` bound as `ASSETS`; custom domains `kaipu.app`, `www.kaipu.app` (`:18-21`); `preview_urls: true` (`:41`, long comment `:22-40`); service binding `API_SERVICE → kaipu-api` (`:42-47`); observability on. |
| `K/apps/console/wrangler.jsonc:1-12` | name `kaipu-console`; compat `2025-10-04`; `workers_dev: false`; `preview_urls: false`; assets `ASSETS`; custom domain `console.kaipu.app`; service binding `API_SERVICE → kaipu-api`. |

Resources that exist but are **not** in any config file (managed by hand / dashboard): R2 buckets
`kaipu-bucket` (public via custom domain `updates.kaipu.app`, set under "R2 → Settings → Custom
Domains" — `deployment/ci-cd-pipelines.md:116,141`) and `kaipu-private-bucket` (API reaches it via
S3 API keys, not a binding — `release-api.yml:103-108`); Cloudflare Access on `kaipu-web` preview
URLs (ADR 0007 Consequences); the `kaipu.app` DNS zone. Database is **Neon Postgres**
(`K/packages/infra-db/package.json:29` `@neondatabase/serverless`), outside Cloudflare. No D1, KV,
Durable Objects or Queues exist anywhere in Kaipu.

## 3. Prior decisions and history (what this review would reopen)

**Hub rule.** `H/monorepos/wrangler-env-config.md:25-29`:

> ## Deploy with Wrangler, not a parallel deploy path
> The apps ship through `wrangler deploy` (CI uses `cloudflare/wrangler-action`). Don't introduce a
> second, redundant deploy path (e.g. an `alchemy.run.ts` orchestrator) alongside it — two paths that
> both "deploy the Worker" drift and one silently wins.

- Written in hub commit `3b1886f` (2026-07-20, "docs: add MVP-first workflow and wrangler/env config
  conventions"); lines 25-29 unchanged since (`git blame`). Commit body: it captures knowledge "that
  previously only lived in template/project CLAUDE.md files". It is the only `alchemy` hit in the hub.
- Its origin: monorepo-template commit `910afa5` (2026-07-11, PR #5, "chore: remove unused Alchemy,
  modernize & pin Wrangler"): "CI deploys every app via cloudflare/wrangler-action (wrangler deploy),
  so the parallel Alchemy path in web-elysia/web-hono was dead weight." It deleted `alchemy.run.ts`
  (x2), "Drop the `.alchemy` Better-T-Stack marker", and added to CLAUDE.md "deploy with Wrangler not
  Alchemy".
- Reading of the reasoning (fact about the text, not a judgment): the rule forbids a **second,
  parallel** path; the recorded reason is that the Alchemy path was unused/dead, not an evaluation of
  Alchemy's merits. The rule does not address replacing wrangler with Alchemy as the **single** path.
- The hub page is partly stale against Kaipu: it says CI uses `wrangler-action` and Wrangler is
  pinned in the Bun catalog; Kaipu's releases no longer use the action (F3) and its wrangler pin is
  not in the catalog (§2).

**Kaipu history.** `alchemy.run.ts` arrived with the template scaffold (`3a06e76f` Initial commit /
`2f5c99ff` 2026-06-15 "customize monorepo template", which also deleted `apps/web-elysia/alchemy.run.ts`).
The deleted file (`git show 52b45368^:apps/web-hono/alchemy.run.ts`) used `alchemy` `^0.81.2`
(lockfile `0.81.4`), `TanStackStart("web", { bindings: { VITE_SERVER_URL } })`, a password from
`ALCHEMY_PASSWORD`, and `CloudflareStateStore` only when `ENVIRONMENT === "production"` (local
`.alchemy/` files otherwise). Scripts: `alchemy:dev`, `deploy`, `destroy`.

| Commit / doc | Date | What it concluded |
| --- | --- | --- |
| `a5c0ee99` docs(secrets) → `backlog/secrets-and-dependency-audit-handoff.md` §2 | 2026-09-21 | "`alchemy.run.ts` bypasses Infisical": it called `dotenv` `config({ path: ".env" })`, and `deploy`/`destroy` did not run `env:pull`, "So `bun run deploy` ships whatever stale `.env` is on disk". Marked **Closed (checked 2026-10-05)**. |
| `b61ca5e3` docs(backlog) → `backlog/deploy-environments.md:31-35` | 2026-10-01 | "An unused second mechanism (`alchemy.run.ts` …) sat beside it, called by no workflow, and was removed — it was already making readers believe the app deployed through Alchemy." |
| `52b45368` chore(web): delete Alchemy | 2026-10-01 | "No workflow called any of them … that is the only path production has ever taken. A second, unused deployment mechanism is not neutral … a design turn concluded the app deployed through Alchemy and planned around a migration that did not need to happen." Local `deploy`/`destroy` not replaced because "a laptop script that deploys straight to production would route around that gate" (ADR 0004). Removed 311 lockfile lines. |
| `5d73a471` feat(ci) preview-web; `1927695e` fix(ci) | 2026-10-01 | Dropped `ENVIRONMENT`: "It existed for Alchemy's state store, Alchemy is deleted" (`preview-web.yml:115-117`). |
| `f4258ac0` `.hermes.md:15` | 2026-10-04 | Still says "Web … deployed with alchemy" — **stale**. |
| `apps/documentation/.../commands.md:95` | — | Still says "`bun run deploy` / `bun run destroy` (Alchemy)" — **stale**. |
| `.claude/env-management-proposal.md:10`, `specs/2026-06-15-dotenvx-env-management-proposal.md:32` | 2026-06-15 | "Vite `import.meta.env` + Alchemy bindings" — historical, pre-removal. |
| `.gitignore:59`, `apps/server-hono/.gitignore:19` | — | `.alchemy` ignore entries left over. |

**Was Alchemy ever tried?** No evidence it ever deployed anything: the deletion commit asserts
`release-web.yml` "has always" used wrangler, and no commit or doc records an Alchemy deploy or an
Alchemy failure. It was never evaluated; it was scaffold residue removed as dead and misleading.

**Related ADRs in Kaipu** (`K/apps/documentation/src/content/docs/architecture/decisions/`):
- `0007-prod-previews-via-version-urls.mdx` (Accepted, 2026-10-01): previews are Version URLs via
  manual `versions upload`; "A persistent development environment is deliberately not built. When
  one becomes necessary, it is a **separate Worker** (own name, own bindings…)". Reopens when a
  pre-production need **writes**.
- `0004-local-first-release-verification.mdx` (Superseded by 0010 → 0027): publish only after
  `verify` on the tagged SHA. `0027-verify-release-candidates-only.mdx` (Accepted).
- `0026-zod-is-the-env-schema-no-varlock.mdx` (Proposed, not on main): Zod in `@kaipu/infra-env` is
  the only env schema; rejects varlock.
- `backlog/deploy-environments.md:119-127` reopen triggers for a persistent env: second tester,
  fixed-address Lighthouse series, user count, API needing pre-prod. Owner answers (2026-10-01):
  kaipu.app has real users, "fewer than 10"; intermediate env wanted long term "as a need for new
  products, not only this one".

**Effect in Kaipu:** no `package.json` in the repo depends on `effect`; `bun.lock` has no `effect`
entry. Hub stance (`H/effect/adoption-strategy.md`, `why-effect.md`): adopt Effect by "reach",
default level 1 inside one leaf package; "Adopting it because it is popular produces exactly the
half-adopted state … the most expensive place to be" (`why-effect.md:152-155`); not worth it for "a
script, a one-off" (`:142`). Hub ADR on Effect: `H/architecture/decisions/adr-0001-effect-use-cases.md`.
The hub says nothing about Effect in tooling/IaC.


## 4. Alchemy today

Source: web research run 2026-11-02 (primary sources fetched: npm registry, GitHub repo/API, alchemy.run docs).
Tags: **[V]** verified from source/docs, **[B]** blog/marketing claim, **[?]** could not verify.

**Release state**
- [V] npm `alchemy` dist-tag `latest` = `2.0.0-beta.84` (2026-10-24); `next` = `2.0.0-beta.72`. A plain `npm i alchemy` / `bun add alchemy` installs the beta. `2.0.0-beta.1` published 2026-04-13. License Apache-2.0. https://registry.npmjs.org/alchemy
- [V] Cadence: 29 betas between beta.56 (2026-06-17) and beta.84 (2026-10-24), about one every 4 days; all marked prerelease on GitHub. https://github.com/alchemy-run/alchemy/releases
- [V] v2 README: "alchemy v2 is in beta. Expect breaking changes." https://github.com/alchemy-run/alchemy
- [V] v1 (async/await, the line Kaipu had at `0.81.4`) continues as 0.x; last release `0.94.0` (2026-08-01), repo `alchemy-run/alchemy-async` (formerly `sam-goodwin/alchemy`, created 2025-01-17, ~2.2k stars). v2 (Effect rewrite) repo `alchemy-run/alchemy`, formerly `alchemy-effect`, created 2025-10-22, ~1.8k stars. https://github.com/alchemy-run/alchemy-async
- [V] ~246 open issues on the v2 repo page (API reports 453 incl. PRs); ~142 contributors (inferred from API pagination).
- [V] Company: site footer "© 2026 Functionless Corp."; npm maintainer Sam Goodwin; CI runners sponsored by Blacksmith. https://alchemy.run
- [?] Funding; a dated formal announcement of the Effect rewrite (only repo creation and beta.1 dates found).
- [V] v1→v2 migration: v1 state is not compatible; v2 starts from empty state; pin v2 resource names to v1's `{app}-{id}-{stage}` and run `alchemy deploy --adopt`; do not run `destroy` from the v1 project afterwards. https://alchemy.run/migrating-from-v1
- [V] Open proposal #2764 to cap `effect` peer ranges because "effect's unstable APIs can break in minor releases".

**Cloudflare coverage** [V] (https://alchemy.run/llms.txt): Workers, Static Assets / framework sites (Vite, TanStack-style Vite apps, Astro, OpenNext…), Durable Objects, Queues, KV, D1 (with migrations), R2, Hyperdrive, custom domains and routes (https://alchemy.run/cloudflare/networking/custom-domains), Zones/DNS, Secrets Store, Containers, Workflows, Access, Turnstile, Vectorize; docs for gradual deployments and Worker Previews.

**Secrets** [V]: read via `effect/Config` (`Config.Redacted`) and bound as `secret_text`; optional Doppler/Infisical secret providers. https://alchemy.run/environments/secrets · https://alchemy.run/environments/secret-providers. [?] whether v2 still uses `ALCHEMY_PASSWORD` (v1 vocabulary; not in v2 docs).

**State** [V] (https://alchemy.run/state-store): default local files `.alchemy/state/<Stack>/<stage>/*.json`; remote `Cloudflare.state()` = a Worker + Durable Object (SQLite) named `alchemy-state-store`, token and encryption key in the account's Secrets Store, encrypted at rest. In CI (`CI=true`) it refuses to bootstrap; run `alchemy state bootstrap --backend cloudflare` once beforehand. AWS/GCP/custom stores also exist.

**Stages / previews** [V] (https://alchemy.run/environments/ci): a stage is an isolated stack instance with its own state and physical names. Documented CI pattern: deploy `pr-<n>` on PRs, `prod` from main, `alchemy destroy --stage pr-<n>` on PR close; PR-comment resource for URLs. Worker Previews documented with isolated DO state: https://alchemy.run/cloudflare/compute/previews

**wrangler.jsonc** [V/?]: v1 had a `WranglerJson` resource (alchemy-async issues #145, #558, #739). v2 docs describe "wrangler-free" dev and "no Wrangler config"; no v2 wrangler.json generator found [?]. In v2, Alchemy replaces `wrangler.jsonc` as the config source rather than reading it.

**Adoption** [V] (https://alchemy.run/cli/adopting-resources): with no prior state, the engine looks up by physical name; silently adopts if it can prove this stack created it; otherwise fails with `OwnedBySomeoneElse` unless `--adopt`, which takes the resource over **and reconciles it to the declared config**. D1 databases migrated by `wrangler d1 migrations apply` / drizzle-kit adopt automatically (https://alchemy.run/cloudflare/data/d1). [?] data preservation in every adopt case — docs imply no recreate; not tested.

**Destroy semantics** [V] (https://alchemy.run/infrastructure-as-code/resource-lifecycle): `RemovalPolicy` default `destroy`; only `Cloudflare.Zone` and `GitHub.Repository` named as defaulting to `retain`. Removing a resource from code, replacing it, or `alchemy destroy` deletes the cloud object; changing a logical ID plans create-new-then-delete-old. `retain` does not stop provider-side cascades (e.g. emptying a bucket).

**Known issues** [V] (repo `alchemy-run/alchemy` unless noted):
- #2604 (closed 2026-10-27): failed existence check on `Cloudflare.state()` read as "missing"; bootstrap overwrote the live store's secrets, making all entries unreadable. Affected `2.0.0-beta.83`.
- #1194 (open): Cloudflare state store is an account-wide singleton whose contract version is compiled into each CLI; version mismatch can redeploy it → lockstep upgrades across stacks.
- #2805 (state store cannot use a different account than providers); #2368 (credential cache scoped to profile).
- #1008 (open, beta.65): persistent 401 during concurrent `--adopt` deploy in GitHub Actions.
- #1785 (open): `Cloudflare.Container` replace with unchanged name adopts then deletes the live resource.
- Secrets: #2800 (secret dropped from `env` neither planned nor deleted); #2757 (secrets written outside Alchemy lost across a script upload); #2816 (plan diff prints previous plaintext of a now-`Redacted` prop); #1138 (plan fails when secret env vars absent); #2073 closed (`AWS.state()` logged secrets at debug).
- AWS-only: #1967 closed (concurrent deploys corrupted state), #2751 open.
- [?] No dedicated Cloudflare API rate-limit issue found.

**Relationship to Cloudflare** [?]: no acquisition, partnership, or Cloudflare-docs adoption found. [B] Better-T-Stack (the scaffold Kaipu's template came from) documents an Alchemy-for-Cloudflare guide: https://www.better-t-stack.dev/docs/guides/cloudflare-alchemy (title seen in search only).

## 5. Alternatives in use (2026)

- **wrangler (status quo).** [V] wrangler latest `4.156.1` (Kaipu CI pins `4.148.0`, workspace `4.63.0`); `cloudflare/wrangler-action` latest `v4.1.3` (2026-09-24) (Kaipu preview uses `v3.14.1`). Cloudflare's first-party tool; config in `wrangler.jsonc`; no state file — the Cloudflare account is the state. https://registry.npmjs.org/wrangler · https://github.com/cloudflare/wrangler-action
- **Wrangler auto-provisioning.** [V] On deploy, Wrangler creates KV, R2, D1, Queues, Dispatch Namespaces (and some newer types) when bindings omit IDs, writing IDs back to config (not written back under dashboard git deploys). https://developers.cloudflare.com/workers/wrangler/configuration/
- **Cloudflare Workers Builds.** [V] Git-connected builds; non-production branches create Previews with a Preview URL and optional PR comments; GA status not stated on page. Version URLs are not generated for Workers using Durable Objects. https://developers.cloudflare.com/workers/ci-cd/builds/ · https://developers.cloudflare.com/workers/configuration/previews/
- **SST.** [V] repo now `anomalyco/sst`, ~26.4k stars, v4.17.2 (2026-10-08); Cloudflare available via Pulumi/Terraform bridges among 150+ providers, no Cloudflare-specific components listed. https://sst.dev/docs/providers/ · [?] company status 2025-2026.
- **Pulumi Cloudflare provider.** [V] v6.23.2 (2026-10-27), ~155 stars, 73 open issues; general-purpose IaC with Pulumi state backend. https://github.com/pulumi/pulumi-cloudflare
- **Terraform / OpenTofu Cloudflare provider.** [V] v5.28.1 (2026-10-20); v5.0.0 (2025-01-29) was a ground-up rewrite generated from OpenAPI, v4 no longer developed; open v5 issues are mostly perpetual plan diffs (#7155 `cloudflare_worker` observability, #7352, #7449). OpenTofu v1.13.2. https://github.com/cloudflare/terraform-provider-cloudflare/releases/tag/v5.0.0

## 6. Facts both sides may cite

- **F1** Kaipu has three Workers (`kaipu-api`, `kaipu-web`, `kaipu-console`); only api and web have deploy workflows; `kaipu-console` has none (§2).
- **F2** No KV, D1, Durable Objects, Queues, R2 bindings, `vars` or `env` blocks exist in any `wrangler.jsonc`. Declared resources are: Static Assets (web, console), 3 custom domains (`kaipu.app`, `www.kaipu.app`, `console.kaipu.app`), 2 service bindings to `kaipu-api`, `preview_urls`/`workers_dev` flags, observability (§2 table).
- **F3** Release deploys do **not** use `cloudflare/wrangler-action` since `8f14a05c` (2026-10-19): they call `scripts/deploy-worker.sh` (pinned wrangler `4.148.0` by absolute path, `deploy --secrets-file`). Reason recorded: the action resolved the app's local wrangler 4.63.0 via npx, and `api-v0.5.0` failed with "Unknown arguments: secrets-file" (`deploy-worker.sh:9-13`, `deployment/ci-cd-pipelines.md:95-100`). Only `preview-web.yml` still uses the action (`v3.14.1`, `versions upload`).
- **F4** Secrets reach Workers inside the same deploy via a `jq`-written `worker-secrets.json` from GitHub `production` Environment values; empty values are dropped so a missing GitHub secret cannot blank production (`release-api.yml:84-125`; hub `worker-secrets-with-deploy.md`). The rule exists because `secret put`/action `secrets:` failed after a preview upload (`web-v0.9.0`, 2026-10-06; `release-web.yml:96-103`).
- **F5** Production secrets still come from GitHub Secrets, not Infisical (`deployment/secrets-layout.md:19`); Infisical has only a `dev` environment.
- **F6** Every publish is gated: `needs: verify` on the tagged SHA (ADR 0004 → 0027); release-candidate PRs dry-run the exact deploy script (`pr-checks.yml:304-312`). Tags come from release-please.
- **F7** Resources outside any config: R2 buckets `kaipu-bucket` (public via `updates.kaipu.app`, set in dashboard) and `kaipu-private-bucket` (used via S3 keys, not bindings); Cloudflare Access on preview URLs; DNS zone `kaipu.app`. Database is Neon Postgres (outside Cloudflare).
- **F8** There is no staging/dev environment. ADR 0007 (Accepted 2026-10-01) chose Version URLs for pre-release looks and says a future dev environment is "a **separate Worker** (own name, own bindings…)", reopened when a pre-prod need writes. Owner (2026-10-01): <10 real users; an intermediate env is wanted long-term "as a need for new products".
- **F9** Hub rule `H/monorepos/wrangler-env-config.md:25-29` (commit `3b1886f`, 2026-07-20) forbids a **second, parallel** deploy path such as `alchemy.run.ts`; it originated in template commit `910afa5` (2026-07-11) which removed Alchemy as "dead weight" because CI deployed via wrangler. Neither text evaluates Alchemy as a sole path.
- **F10** Kaipu's `alchemy.run.ts` (v1 `^0.81.2`, `TanStackStart`, `CloudflareStateStore` in production) was template scaffold residue; no workflow ever called it; deleted in `52b45368` (2026-10-01) because it misled readers and a design session "planned around a migration that did not need to happen". Earlier audit (2026-09-21) found it bypassed Infisical by loading `.env` directly.
- **F11** Stale docs still claim Alchemy: `.hermes.md:15`, `apps/documentation/.../commands.md:95`; leftover `.alchemy` lines in `.gitignore:59`, `apps/server-hono/.gitignore:19`.
- **F12** Kaipu has no `effect` dependency anywhere. Hub Effect guidance: adopt by reach, start at level 1 in a leaf; popularity-driven adoption leads to the "half-adopted" trap; not worth it for scripts (`H/effect/why-effect.md:137-155`).
- **F13** Wrangler pinning is split: workspace `4.63.0` in `server-hono` and `console` package.json (not in the Bun catalog, contrary to the hub rule); CI `4.148.0`; latest `4.156.1`. `wrangler preview` needs ~4.146+ (`backlog/deploy-environments.md:100-102`).
- **F14** Alchemy v2 is beta (`2.0.0-beta.84`, 2026-10-24), README "Expect breaking changes", ~1 release / 4 days; it is the npm `latest`. v1 (`0.94.0`, 2026-08-01) continues separately; v1→v2 state is incompatible.
- **F15** Alchemy covers every Cloudflare resource Kaipu uses or might use (Workers, assets, custom domains, routes, R2, D1, KV, DO, Queues, Access, DNS, Secrets Store).
- **F16** Alchemy's default `RemovalPolicy` is `destroy` for data resources (D1/KV/R2); removing code or `alchemy destroy` deletes them; `--adopt` reconciles adopted resources to declared config.
- **F17** Alchemy's Cloudflare state store is an account-wide Worker+DO singleton; #2604 (fixed 2026-10-27, beta.83) overwrote store secrets making all state unreadable; #1194 (open) forces lockstep CLI versions across stacks; CI cannot bootstrap it.
- **F18** Alchemy secret-handling issues open as of research: #2800, #2757, #2816, #1138; #1008 (401 on concurrent `--adopt` in GitHub Actions).
- **F19** Alchemy natively models per-PR stages (`pr-<n>` deploy / destroy), with isolated state and physical names — including service bindings resolved per stage [mechanism documented; per-stage service-binding resolution is implied by isolated physical names, not separately verified].
- **F20** Wrangler auto-provisions KV/R2/D1/Queues on deploy and Workers Builds gives per-branch Preview URLs; Version URLs are unavailable for Workers with Durable Objects.
- **F21** Terraform Cloudflare provider v5 is a 2025 OpenAPI-generated rewrite with recurring perpetual-diff issues; Pulumi Cloudflare v6.23.2 is active; SST reaches Cloudflare only through bridges.
- **F22** Hub `infra/deploy-environments.md:84-93`: bindings are not inherited between Wrangler environments; service bindings bind by Worker name, so a staging front end on a shared binding writes to the production API.

## 7. Open uncertainties

- **U1** Can Alchemy v2 adopt the existing `kaipu-web`/`kaipu-api` Workers and their custom domains without downtime or DNS/cert re-issuance? Not tested.
- **U2** Does `--adopt` of the existing R2 buckets (and their dashboard-set custom domain `updates.kaipu.app`, Access policies) reconcile away settings Alchemy's declaration omits?
- **U3** How would Alchemy carry the "secrets ship in the deploy, additive, empty-dropped" invariant (F4)? #2800/#2757 suggest secret lifecycle differs; behavior against a Worker with a pending undeployed version (preview upload) is unknown.
- **U4** Does Alchemy v2 produce or consume a `wrangler.jsonc`, so `wrangler dev`, `wrangler types`, `versions upload` and the dry-run gate keep working? v2 docs say "no Wrangler config"; no generator confirmed.
- **U5** Where would state live, and who bootstraps it? `Cloudflare.state()` cannot bootstrap in CI; local state does not work across CI runs; S3/GCP would add a provider.
- **U6** Does Alchemy's TanStack Start / Vite integration build `apps/web-hono` identically to the current `bun run build` + wrangler bundle (VITE_* build-time vars, `@tanstack/react-start/server-entry`)?
- **U7** When does v2 leave beta, and how often have betas broken Cloudflare users? Only issue anecdotes found; no stability statement.
- **U8** Would Alchemy pull `effect` into Kaipu's runtime bundle or only into the deploy script? Presumably deploy-time only; not verified. Relevant to F12.
- **U9** Cloudflare API rate-limit behavior of Alchemy in CI: no data.
- **U10** Is the eventual dev environment (ADR 0007's separate Worker) likely to need resources wrangler auto-provisioning does not cover (Access, DNS, per-stage service-binding targets)? Depends on product plans not recorded.
- **U11** Funding/backing of Functionless Corp. and bus factor beyond Sam Goodwin: not found.
- **U12** Whether the console Worker is meant to be deployed at all (no workflow, not a release-please component).

## 8. Evidence index

**Hub** (`H/` = `~/Documents/Workspace/Personal/general-knowledge`): `monorepos/wrangler-env-config.md` (+ `git show 3b1886f`, `git blame -L 20,35`); `monorepos/ci-cd-pipelines.md`; `monorepos/worker-secrets-with-deploy.md`; `monorepos/release-gated-verification.md`; `infra/deploy-environments.md`; `infra/README.md`; `infra/custom-domain-migration.md` (grep only); `infra/infisical-secrets.md` (head + grep); `infra/varlock-evaluation.md` (head); `stacks/service-only-hono.md`; `api/hono.md`, `web/local-preview.md`, `api/decisions/adr-0002-orpc-hono-cloudflare.md` (grep only); `effect/README.md`, `effect/why-effect.md`, `effect/adoption-strategy.md`; `conventions/adr-pattern.md`. No hub page mentions Pulumi, SST, Terraform or IaC tooling.

**Template** (`~/Documents/Workspace/Personal/monorepo-template`): commit `910afa5` (2026-07-11).

**Kaipu** (`K/` = `~/Documents/Workspace/Personal/kaipu-record-monorepo`, `c01796bb`):
- `.github/workflows/release-api.yml`, `release-web.yml`, `preview-web.yml`, `release-please.yml`, `pr-checks.yml:295-315`, `ci-api.yml` (head); `release-please-config.json`
- `scripts/deploy-worker.sh` (commit `8f14a05c`)
- `apps/{server-hono,web-hono,console}/wrangler.jsonc`; `apps/*/package.json`; root `package.json`; `packages/infra-db/package.json`
- Commits: `52b45368` (+ `git show 52b45368^:apps/web-hono/alchemy.run.ts`), `a5c0ee99`, `b61ca5e3`, `5d73a471`, `1927695e`, `f4258ac0`, `2f5c99ff`
- Docs (`apps/documentation/src/content/docs/`): `backlog/deploy-environments.md`, `backlog/secrets-and-dependency-audit-handoff.md`, `commands.md:88-100`, `deployment/ci-cd-pipelines.md`, `deployment/secrets-layout.md`, `backlog/r2-storage-architecture.md` (grep), `architecture/decisions/0007-prod-previews-via-version-urls.mdx`, `0004-…`, `0010-…`, `0026-…`, `0027-…` (status lines); `.hermes.md`.
- Other deploy/infra docs present but not read: `deployment/pr-checks.md`, `deployment/google-oauth.md`, `backlog/production-cloud-security.md`, `backlog/ci-cost-handoff.md`, `backend/environment-variables.mdx`.

**Web** (fetched 2026-11-02): https://registry.npmjs.org/alchemy · https://github.com/alchemy-run/alchemy · https://github.com/alchemy-run/alchemy/releases · https://github.com/alchemy-run/alchemy-async · https://alchemy.run · https://alchemy.run/llms.txt · https://alchemy.run/migrating-from-v1 · https://alchemy.run/state-store · https://alchemy.run/environments/ci · https://alchemy.run/environments/secrets · https://alchemy.run/environments/secret-providers · https://alchemy.run/cloudflare/compute/previews · https://alchemy.run/cloudflare/networking/custom-domains · https://alchemy.run/cloudflare/data/d1 · https://alchemy.run/cli/adopting-resources · https://alchemy.run/infrastructure-as-code/resource-lifecycle · GitHub issues `alchemy-run/alchemy` #2604 #1194 #2805 #2368 #1967 #2751 #1008 #1785 #2800 #2757 #2816 #1138 #2073 #2764 · https://registry.npmjs.org/wrangler · https://github.com/cloudflare/wrangler-action · https://developers.cloudflare.com/workers/wrangler/configuration/ · https://developers.cloudflare.com/workers/ci-cd/builds/ · https://developers.cloudflare.com/workers/configuration/previews/ · https://github.com/anomalyco/sst · https://sst.dev/docs/providers/ · https://github.com/pulumi/pulumi-cloudflare · https://github.com/cloudflare/terraform-provider-cloudflare/releases/tag/v5.0.0 · https://github.com/opentofu/opentofu · https://www.better-t-stack.dev/docs/guides/cloudflare-alchemy (search result only).
