# Alchemy: evaluated, not adopted

> Status: **evaluated and set aside** (2026-11-02). No product deploys through
> Alchemy. `kaipu-record-monorepo` carried an unused `alchemy.run.ts` from the
> template scaffold and deleted it on 2026-10-01; the owner also reports an
> earlier attempt in another project that did not work out (not recorded in any
> repository). This page records what a real adoption would cost, so the
> question is answered from evidence next time instead of from affection for
> the project. Written after an adversarial review (one agent for, one against,
> both working from the same neutral context package) and a source check of the
> claims that decide it.

**Documentation boundary:** how Workers ship today is in
[wrangler-env-config.md](../monorepos/wrangler-env-config.md) and
[worker-secrets-with-deploy.md](../monorepos/worker-secrets-with-deploy.md);
whether a pre-production tier is worth building is in
[deploy-environments.md](./deploy-environments.md). This page only evaluates
Alchemy against that baseline.

## What Alchemy is, as of 2026-11-02

[Alchemy](https://alchemy.run) is infrastructure-as-code in plain TypeScript:
one `alchemy.run.ts` declares Workers, buckets, domains and bindings, the CLI
diffs that against a state store and reconciles the Cloudflare account. The v2
line is a rewrite on [Effect](../effect/README.md) by Functionless Corp. (Sam
Goodwin), Apache-2.0.

Facts checked against npm, GitHub and the docs on 2026-11-02:

- `2.0.0-beta.84` (2026-10-24) is npm `latest`, so a plain `bun add alchemy`
  installs the beta. About one beta every four days since June. README: "alchemy
  v2 is in beta. Expect breaking changes." The v1 line (`0.x`, the one the
  template shipped) continues separately and its state is incompatible with v2.
- Cloudflare coverage is complete for anything a small product uses: Workers,
  Static Assets, custom domains and routes, R2, D1, KV, Durable Objects, Queues,
  Access, DNS zones, Secrets Store.
- Stages are first class: `--stage pr-12` gives isolated physical names and
  state, and the documented CI pattern creates a stage per PR and destroys it
  on close.
- State lives in local files by default, or in a remote backend:
  `Cloudflare.state()` (a Worker + Durable Object, one per account),
  `postgresState()` (any Postgres you already run, with a session advisory lock
  per stack and stage; needs `pg` and `@effect/sql-pg`), AWS S3, GCP, or a
  custom Effect `Layer`.
- `RemovalPolicy` defaults to `destroy`. Removing a resource from code, renaming
  its logical ID, or running `alchemy destroy` deletes the cloud object. Only
  zones and GitHub repositories default to `retain`.
- `--adopt` takes over a resource Alchemy did not create **and reconciles it to
  the declared config**. Settings the declaration omits are not guaranteed to
  survive.

## What it would cost

The list a serious adoption has to pay, in the order it would hurt. Each item
names the evidence; "unverified" means nobody has run it.

| # | Cost | Why it exists | Evidence |
| --- | --- | --- | --- |
| 1 | **Re-proving the secrets invariant.** Secrets must ship inside the same deploy, additively, with empty values dropped, and must survive a `versions upload` that was never deployed. | That rule was earned by a production incident (`web-v0.9.0`, 2026-10-06). Alchemy's secret lifecycle has four open issues: a secret dropped from `env` is neither planned nor deleted (#2800), secrets written outside Alchemy are lost across a script upload (#2757), a plan diff prints the previous plaintext of a now-redacted value (#2816), plan fails when secret env vars are absent (#1138). The pending-version case is **unverified**. | [worker-secrets-with-deploy.md](../monorepos/worker-secrets-with-deploy.md); issues on `alchemy-run/alchemy` |
| 2 | **Operating a state backend.** Someone owns bootstrap, credentials, recovery and version compatibility. | Wrangler has no state: the account is the state. `Cloudflare.state()` is an account-wide singleton whose contract version is compiled into the CLI, so unrelated stacks must upgrade in lockstep (#1194), and it refuses to bootstrap in CI. A failed existence check in beta.83 overwrote the live store's secrets and made every entry unreadable (#2604, fixed 2026-10-27). `postgresState()` on an existing Neon database sidesteps the singleton, at the price of a deploy that depends on the database being up. | <https://alchemy.run/state-store> |
| 3 | **Adopting live resources without losing their settings.** R2 buckets, a public bucket domain set in the dashboard, Access policies on preview URLs, the DNS zone. | `--adopt` reconciles to the declaration; whether a bucket's dashboard-set custom domain or an Access policy survives an incomplete declaration is **unverified**. Destroy-by-default means a wrong plan on a data resource deletes it unless `retain` was set first. Kaipu has no D1, KV, DO or Queues, so the loud data-loss stories do not apply; the R2 and Access cases do. | <https://alchemy.run/cli/adopting-resources>, <https://alchemy.run/infrastructure-as-code/resource-lifecycle> |
| 4 | **Losing `wrangler.jsonc` as the config source.** `wrangler dev`, `wrangler types`, `versions upload` and the release-candidate dry run all read it. | v2 docs describe "no Wrangler config"; no v2 generator for `wrangler.jsonc` was found. Either a hand-kept parallel config appears (the exact drift the hub rule forbids) or every one of those tools is replaced. **Unverified** whether v2 can emit one. | <https://alchemy.run/llms.txt> |
| 5 | **Build parity for the framework app.** The web Worker is built by `bun run build` with `VITE_*` inputs, then bundled by wrangler. | Alchemy's Vite/TanStack integration builds it its own way. Byte-level parity of `dist/client` and the Worker bundle is **unverified**, as is whether `effect` leaks into the runtime bundle. | context package U6, U8 |
| 6 | **Tracking a breaking-change beta.** Pin `alchemy` and `effect` exactly, and review every bump with a plan that shows zero unintended changes. | One beta every four days, "expect breaking changes", and an open proposal to cap `effect` peer ranges because its unstable APIs break in minors (#2764). A hotfix during a broken beta needs a break-glass path, which during migration means keeping the wrangler script alive. | <https://github.com/alchemy-run/alchemy/releases> |
| 7 | **Reading Effect failures.** When a deploy fails, the trace runs through the fiber runtime and a `Cause` has to be read. | Nobody on these products writes Effect today. This is the hub's known debugging cost, not an adoption of Effect: the runtime is owned by Alchemy's CLI and the boundary never reaches product code, which is below "level 1" in [adoption-strategy.md](../effect/adoption-strategy.md). It becomes real adoption only if a custom resource has to be written. | [why-effect.md](../effect/why-effect.md) |
| 8 | **A transition with two tools.** During any pilot the wrangler script keeps shipping production while Alchemy owns a dev stage. | Not two paths to the same Worker, so it does not violate the hub rule, but it is two tools to understand and it needs a sunset date. Every stale doc that said "deploys through Alchemy" cost a design session once already. | kaipu commit `52b45368` |

What it would buy, stated once so the cost table is not read as a verdict:
every Cloudflare resource declared in one reviewable file with a real
plan step on release-candidate PRs instead of a dry run; a dev or per-PR stage
whose service binding resolves to the stage's own API instead of production,
which is the
[costliest mistake in deploy-environments.md](./deploy-environments.md); and the
deploy tool as a normal lockfile dependency instead of a version pinned by hand
in a shell script and mirrored into CI.

## Direct service connections and R2: what is known and what is not

The specific doubt behind this evaluation was services that talk to each other
directly, and R2.

- **Service bindings.** Alchemy's stage model gives each stage its own physical
  Worker names, and the docs imply a binding declared in code resolves to the
  same stage's Worker. That is the mechanism a dev tier needs. It is implied by
  isolated names, **not separately verified** against a deployed pair.
- **R2 accessed through the S3 API** with account keys, as the API and the
  desktop uploader do, is untouched by any deploy tool. Alchemy only matters for
  R2 when the bucket itself, its public custom domain or a binding is declared.
- **R2 declared and adopted** is the risky case: `destroy` by default, and
  reconciliation of a dashboard-set custom domain is unverified. If a bucket is
  ever adopted, `RemovalPolicy.retain()` goes on it in the same commit, and the
  CI token for the first runs has no R2 delete scope.
- **Access on preview URLs** is a dashboard-managed policy today. Alchemy covers
  the resource type; adopting it without re-creating the policy is unverified.

## Why it was set aside

1. **There is almost no infrastructure to manage.** Three Workers, static
   assets, three custom domains, two service bindings, two buckets reached by
   S3 keys, one Access policy, one zone. No D1, KV, Durable Objects or Queues.
   The database is Neon, outside Cloudflare. An IaC layer earns its state store
   when the resource graph is wide or repeated; here it is neither yet.
2. **The pains in the current pipeline have cheaper fixes.** Wrangler version
   drift (three versions in play) is fixed by pinning once in the catalog, the
   existing rule. A Worker with no deploy workflow needs a product decision, not
   a new engine. Stale docs need editing. None of these is an argument for
   Alchemy; they were just adjacent to it.
3. **The one real benefit is ahead of the recorded need.** A persistent dev
   environment was deferred on purpose (Kaipu ADR 0007) until pre-production
   work needs to write. The owner wants it "long term, as a need for new
   products". When that trigger fires, Alchemy's stage model is the strongest
   answer on the table; until then it is building the environment the ADR
   deferred.
4. **The two risks nobody can bound by design are secrets (cost 1) and
   adoption (cost 3).** Both are testable, neither has been tested, and the
   products have real users.

The hub rule in
[wrangler-env-config.md](../monorepos/wrangler-env-config.md) was not the
reason. It forbids a *second, parallel* deploy path, and it came from deleting
scaffold residue that no workflow ever called. A single-path adoption would not
violate it. Nothing in any repository evaluated Alchemy's merits before this
page.

## What would reopen this

Any one of these is enough to run the pilot below; adoption still needs the
pilot to pass.

- A pre-production tier that **writes** becomes a requirement, or two or more
  writable environments are needed routinely (the ADR 0007 reopen condition).
- Provisioning becomes repeated work: a second product on the same account, a
  recorded drift incident on a dashboard-managed resource, or an Access/DNS/R2
  setup that cannot be reproduced from any record.
- The product grows a resource wrangler does not auto-provision well: Durable
  Objects (which also lose Version URLs), Queues, per-stage Access or DNS.
- Alchemy v2 reaches a stable release with a documented upgrade policy, or a
  pinned beta has run a month of someone else's production deploys without a
  state or secrets incident.

And what would close it harder: no release on `alchemy-run/alchemy` for 90
days; v2 still in beta two years after beta.1 (2028-04-13); or a pilot that
needs a custom Effect resource to cover a setting Kaipu uses.

## If adopted anyway: the pilot and its guardrails

The smallest reversible experiment, touching no production object:

- A stack declaring the API and web Workers with stage-suffixed names
  (`<app>-api-dev`, `<app>-web-dev`), `workers_dev` on, no custom domains, the
  service binding resolved per stage. State in `postgresState()` on a dedicated
  database. `alchemy` and `effect` pinned exactly in the Bun catalog.
- Triggered by `workflow_dispatch` from CI only, with a dedicated GitHub
  Environment holding non-production values. No laptop deploy script: that
  would route around the release gate. Release workflows untouched.
- CI token scoped to Workers; no R2, zone or Access delete permissions.
- Must prove, in order: (1) the secrets sequence that caused the 2026-10-06
  incident passes with every key present and empty values dropped; (2) build
  output matches the wrangler build and no `effect` is in the runtime bundle;
  (3) the dev web Worker calls the dev API, never production; (4) a second
  `alchemy plan` reports no changes; (5) `alchemy destroy --stage dev` removes
  exactly the two Workers.
- Budget: three working days. Any failed step reverts the pilot and the result
  is recorded here and in the product's deploy-environments backlog page.
- If it passes: adopt the production API Worker only after a plan shows zero
  diff, `retain` on every data resource before it is declared, no `destroy` in
  any CI job that can see production, and the wrangler deploy script deleted
  within one release cycle after the web Worker moves, so the second tool has a
  sunset.

## Raw reports

The context package and both advocate briefs are kept verbatim in
[docs/reviews/2026-11-02-alchemy-adversarial-review/](../docs/reviews/2026-11-02-alchemy-adversarial-review/README.md).

## References

- Alchemy: <https://alchemy.run> · state stores <https://alchemy.run/state-store> ·
  CI and stages <https://alchemy.run/environments/ci> · secrets
  <https://alchemy.run/environments/secrets> · adoption
  <https://alchemy.run/cli/adopting-resources> · lifecycle and removal policy
  <https://alchemy.run/infrastructure-as-code/resource-lifecycle> · v1 to v2
  <https://alchemy.run/migrating-from-v1>
- Releases and issues: <https://github.com/alchemy-run/alchemy/releases>; issues
  #2604, #1194, #1008, #2800, #2757, #2816, #1138, #2764 on
  <https://github.com/alchemy-run/alchemy>; v1 line
  <https://github.com/alchemy-run/alchemy-async>
- Cloudflare first-party alternatives: auto-provisioning
  <https://developers.cloudflare.com/workers/wrangler/configuration/> · Workers
  Builds <https://developers.cloudflare.com/workers/ci-cd/builds/> · Version URLs
  and previews <https://developers.cloudflare.com/workers/configuration/previews/>
- Other IaC with Cloudflare support, for completeness: Pulumi
  <https://github.com/pulumi/pulumi-cloudflare> · Terraform/OpenTofu provider
  v5 <https://github.com/cloudflare/terraform-provider-cloudflare> · SST (via
  bridges) <https://sst.dev/docs/providers/>
- Hub pages this builds on: [deploy-environments.md](./deploy-environments.md),
  [wrangler-env-config.md](../monorepos/wrangler-env-config.md),
  [worker-secrets-with-deploy.md](../monorepos/worker-secrets-with-deploy.md),
  [effect/adoption-strategy.md](../effect/adoption-strategy.md),
  [effect/why-effect.md](../effect/why-effect.md)
