# Web Frontend Knowledge

Generalized, product-agnostic notes on building web frontends with **TanStack Start / Router / Query**, a shared UI package, and a typed API client. Patterns are extracted from real monorepo apps and stripped of product-specific naming.

## Contents

| Doc | Summary |
| --- | --- |
| [data-loading.md](./data-loading.md) | Router loaders + TanStack Query: server functions vs isomorphic API client, prefetch patterns, SSR hydration. |
| [server-functions.md](./server-functions.md) | When to use `createServerFn` (web-only cookie work) vs the isomorphic API client, plus authoring rules and gotchas. |
| [web-ui-package.md](./web-ui-package.md) | Shared `web-ui` package: shadcn components, Tailwind v4 theming with OKLCH tokens, and workflow. |
| [bundle-splitting.md](./bundle-splitting.md) | Reduce a large main JS chunk using Vite `manualChunks` for vendor libraries. |
| [local-preview.md](./local-preview.md) | Run a production-equivalent build locally and understand why dev and prod builds differ. |
| [core-web-vitals.md](./core-web-vitals.md) | **Measuring honestly**: never diagnose from the dev server, Lighthouse over a hand-rolled script, read both presets, and the two corrections a localhost run needs (no compression, no edge). The `FCP ≈ LCP` signature, validating the instrument when a number surprises you, and an append-only baseline with the method pinned to it. |
| [tailwind-v4-split-css-cascade.md](./tailwind-v4-split-css-cascade.md) | **Prod-only bug**: responsive layout (`md:`/`lg:` display toggles) breaks in the production build — duplicated Tailwind utilities layer across split stylesheets. Symptom, root cause, and the single-Tailwind-source fix. |
| [pending-navigation.md](./pending-navigation.md) | Global loading bar + per-route `pendingComponent` for navigation feedback. |
