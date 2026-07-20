# Wrangler & env config: single source of truth

_How to keep Cloudflare Workers configuration and environment variables from drifting across a
Turborepo + Bun monorepo. One source of truth per concern, no duplication in `wrangler.jsonc`._

Every app that deploys to Cloudflare Workers carries a `wrangler.jsonc`. The trap is letting that
file accumulate config that already lives elsewhere — env vars, secrets, per-environment blocks —
until the two drift apart and a deploy picks up stale values. These rules keep each concern in
exactly one place.

## Env vars live in `.env`, never in a `vars` block

**Never declare a `vars` / `[vars]` / `[env.*]` block in `wrangler.jsonc`.** Recent Wrangler
versions auto-load `.env` (and `.dev.vars` for local Worker secrets) at dev and build time, so a
`vars` block duplicates config and drifts from `.env` — which is the single source of truth.

- **Local:** each app has its own `.env` (public/build vars) and `.dev.vars` (local Worker
  secrets). Copy the app's `.env.example` and fill it in.
- **Production:** values are injected as Wrangler secrets by `cloudflare/wrangler-action` in CI
  (`secrets:` list + matching `env:` block), not committed anywhere. See
  [ci-cd-pipelines.md](./ci-cd-pipelines.md) for the secret/variable matrix.

## Deploy with Wrangler, not a parallel deploy path

The apps ship through `wrangler deploy` (CI uses `cloudflare/wrangler-action`). Don't introduce a
second, redundant deploy path (e.g. an `alchemy.run.ts` orchestrator) alongside it — two paths that
both "deploy the Worker" drift and one silently wins.

## Pin Wrangler once, in the Bun catalog

Wrangler is pinned in the root `package.json` → `workspaces.catalog.wrangler`, and every app
references it as `"wrangler": "catalog:"`. Bump it in the catalog (one edit) and mirror that version
in the CI step that installs it globally (`bun add -g wrangler@<version>` in the deploy workflow), so
every app and CI resolve the same Wrangler.

## Keep `compatibility_date` current and identical

Keep `compatibility_date` current and **identical across all `wrangler.jsonc` files**. When you bump
it, set today's date and review the Cloudflare compatibility-flags changelog for behavior changes.

## After editing `wrangler.jsonc`, regenerate types

Run `wrangler types` to regenerate `worker-configuration.d.ts` after any `wrangler.jsonc` change
(bindings, services, compatibility date). Stale generated types are a common source of confusing
Worker build errors.
