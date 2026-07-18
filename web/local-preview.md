# Local Preview (Pre-prod)

_How to run a production-equivalent build locally and why dev vs prod builds differ._

## Command

```bash
bun run preview:local
```

Runs from the monorepo root. The exact steps depend on **how the web app gets its
backend** — the two variants differ only in whether a local API process is needed.

### Variant A — external API worker (Hono/oRPC/Elysia on Cloudflare Workers)

The web app calls a separate API worker, so the script boots it first:

1. Start the API app (`wrangler dev`, e.g. port 3000) in the background
2. Build `@repo/web-ui` — compile the component library to `dist/`
3. Build the web app — full Vite production bundle
4. Serve the web app with `vite preview` on `localhost:4173`

Ctrl+C kills the API process group (wrangler + all its children) cleanly.

### Variant B — Convex backend (no local API step)

Convex runs its **own** dev server (`convex dev`) separately and the web app talks
to a Convex **deployment URL** read from `.env` at build time — there is no API
worker to boot. So the script is just build → build → serve:

```bash
#!/usr/bin/env bash
set -e
ROOT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"

# Prod resolves @scope/web-ui to ./dist (the `import` condition), NOT ./src
# (the `development` condition dev uses) — build the library first.
bun run --cwd "$ROOT_DIR/packages/web-ui" build
bun run --cwd "$ROOT_DIR/apps/web" build      # full Vite production bundle
bun run --cwd "$ROOT_DIR/apps/web" serve       # vite preview on :4173
```

The prod build reads the Convex URLs from `apps/web/.env`; pointing at your running
Convex dev deployment is fine for a styling/bundling check (the goal is the
frontend, not the backend).

## Why dev and prod builds differ

### CSS bundle splitting

In dev, Vite serves a single unified stylesheet. In prod, Vite splits CSS across multiple files loaded in a specific order:

- `index-*.css` — compiled from `apps/web/src/index.css` (includes `@repo/web-ui/styles.css`)
- `main-*.css` — bundled with the router chunk, injected by TanStack Start

This split matters because **Tailwind v4 uses CSS logical properties**: `px-2.5` compiles to `padding-inline`, not `padding-left`/`padding-right`. If `px-2.5` (from a component's base class) lands in a stylesheet that loads _after_ a physical `pl-9` override, `padding-inline` wins the cascade and the override has no effect. This class of bug is invisible in dev but visible in prod.

### Workspace packages resolve differently

In dev, Vite resolves `@repo/web-ui` through `resolve.conditions: ["development"]` — it reads TypeScript source directly. In prod, it reads the built `dist/` output. If `packages/web-ui/` has uncommitted source changes, those changes won't appear in the production build until `bun run build` is run inside `packages/web-ui/`. `preview:local` handles this automatically by building the library first.

### Minification and tree-shaking

Prod runs Rollup with full minification and tree-shaking. Dead code that's harmless in dev can expose import errors in prod (e.g., `cloudflare:workers` virtual modules leaking into the client bundle).

## When to use it

Use `preview:local` before opening a PR whenever:

- You changed anything in `packages/web-ui/` (CSS, component base classes)
- You're working on layout/styling and want to confirm it matches prod
- A visual bug only reproduces in prod and you need a fast iteration loop

## See also

The canonical prod-only styling bug this catches:
[tailwind-v4-split-css-cascade](./tailwind-v4-split-css-cascade.md) — responsive
`md:`/`lg:` display toggles break in prod because a duplicated Tailwind utilities layer
loads last and overrides them. `preview:local` is how you reproduce it.
