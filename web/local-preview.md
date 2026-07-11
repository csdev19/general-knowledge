# Local Preview (Pre-prod)

_How to run a production-equivalent build locally and why dev vs prod builds differ._

## Command

```bash
bun run preview:local
```

Runs from the monorepo root. A typical `preview:local` script does, in order:

1. Starts the API app (`wrangler dev`, e.g. port 3000) in the background
2. Builds `@repo/web-ui` — compiles the component library to `dist/`
3. Builds the web app — full Vite production bundle
4. Serves the web app with `vite preview` on `localhost:4173`

Ctrl+C kills the API process group (wrangler + all its children) cleanly.

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
