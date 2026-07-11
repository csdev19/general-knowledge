# Bundle Splitting

_How to reduce a large main JS chunk with Vite `manualChunks`._

## Problem

`bun run build` on a TanStack Start web app can produce a single `main-*.js` chunk of ~1,080 kB (gzip: ~333 kB). Vite's default behavior puts all non-async route code and shared dependencies into this one chunk. The build already warns about it.

Typical contributors identified from the build output:

| Chunk          | Size (min) |
| -------------- | ---------- |
| `main-*.js`    | 1,079 kB   |
| `useForm-*.js` | 201 kB     |
| `main-*.css`   | 412 kB     |

## Fix

Add `manualChunks` to `apps/web/vite.config.ts` inside `build.rollupOptions.output`:

```ts
build: {
  rollupOptions: {
    external: ["cloudflare:workers"],
    output: {
      manualChunks: {
        "vendor-react": ["react", "react-dom"],
        "vendor-tanstack": [
          "@tanstack/react-query",
          "@tanstack/react-router",
          "@tanstack/react-start",
        ],
        "vendor-forms": ["react-hook-form", "@hookform/resolvers", "zod"],
        "vendor-api": ["@orpc/client", "@orpc/openapi-client", "@orpc/react-query"],
      },
    },
  },
},
```

## Caveats

- The `tanstackStart` plugin owns route-level splitting for TanStack Start — `manualChunks` only splits vendor libraries, not route components. Route-level lazy loading is handled automatically by TanStack Router's file-based routing when the `@tanstack/react-start` Vite plugin is configured correctly; no extra work needed there.
- On a Cloudflare Workers deployment, the `cloudflare:workers` unresolved import warning during the API app build is **expected and harmless** — it's a Cloudflare runtime virtual module unavailable at build time. It is already correctly listed in `build.rollupOptions.external`.
- Verify chunk names after implementing: Vite may rename chunks based on hashing. Run `bun run build` and inspect `dist/client/assets/` to confirm vendor chunks are present.

## Expected Result

After applying, the main chunk should drop to ~400–600 kB with separate `vendor-react`, `vendor-tanstack`, `vendor-forms`, and `vendor-api` chunks that are cached independently by the browser.

## Files to Change

| File                      | Change                                                 |
| ------------------------- | ------------------------------------------------------ |
| `apps/web/vite.config.ts` | Add `output.manualChunks` inside `build.rollupOptions` |
