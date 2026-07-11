# Shared package build & consumption strategy

_How workspace packages are consumed as source in dev (HMR, no per-package dev server) and built for prod, including the Node/desktop main-process gotcha and the tsdown `devExports` pattern._

How a `@scope/*` workspace package is authored so that:

- **Dev** consumes the package's **TypeScript source directly** — full HMR, no per-package dev
  server, no build step in the loop.
- **Prod** is just "build everything" — `turbo run build` compiles each package once, apps bundle
  from source, and the package ships a real `dist/` for anyone who publishes/consumes it as JS.

This is the pattern behind packages like `@scope/domain`, `@scope/web-ui`, and `@scope/i18n`. The
`@scope/i18n` package is the worked example here because it is the first package consumed by a
**Node runtime** (a desktop app's main process), which surfaces a gotcha the bundler-only packages
never hit.

## The core idea: source in dev, dist for publish

A bundler-consumed package points its `exports` at **source** in the local workspace, and at the
built **dist** only under `publishConfig` (which npm swaps in on publish). Tools that understand the
`development`/default conditions read source; a published tarball reads dist.

`tsdown` generates both automatically from `exports.devExports`:

```ts
// packages/i18n/tsdown.config.ts
import { defineConfig } from "tsdown";

export default defineConfig({
  entry: ["src/index.ts", "src/web.ts", "src/main.ts"],
  format: "esm",
  dts: true,
  clean: true,
  noExternal: ["use-intl"],
  exports: {
    devExports: true, // local `exports` → src; `publishConfig.exports` → dist
    // Non-entry assets (raw JSON) aren't tsdown entries, so re-add their subpaths:
    customExports(exports) {
      exports["./messages/es"] = "./messages/es.json";
      exports["./messages/en"] = "./messages/en.json";
      return exports;
    },
  },
});
```

Running `bun run build` in the package **rewrites `package.json`** to:

```jsonc
{
  "exports": {                      // local monorepo: source
    ".": "./src/index.ts",
    "./main": "./src/main.ts",
    "./messages/es": "./messages/es.json"
  },
  "publishConfig": {                // npm publish: built dist
    "exports": {
      ".": "./dist/index.mjs",
      "./main": "./dist/main.mjs",
      "./messages/es": "./messages/es.json"
    }
  }
}
```

**Never hand-edit the `exports` / `publishConfig.exports` fields** — `tsdown` owns them. Change the
config and re-run `bun run build`.

`dist/` is **git-ignored** (like every other package) — it is a build artifact, produced by
`turbo run build` (whose `build` task `dependsOn: ["^build"]`, so a package's dist is built before
anything that depends on it).

### Why this is nice

- **Dev = source binding.** The bundler (e.g. Vite, in the web app and a desktop renderer) resolves
  `@scope/i18n` straight to `src/*.ts`, so editing the package hot-reloads in the consuming app with
  zero extra process. No "watch + rebuild the package" server in the dev loop.
- **Prod = build everything.** `turbo run build` compiles each package once; caching makes repeats
  instant. Apps bundle the source at their own build time.

## The Node main-process gotcha

The renderer and the web app are **bundler** targets (e.g. Vite) — they compile TypeScript, so
consuming a package's `src` is free. A desktop **main process is Node**, and Node-targeted build
tools (like electron-vite, or Vite's SSR build) **externalize** `node_modules` dependencies by
default. So `@scope/i18n` was left as a runtime `import`, and at launch Node tried to load the
package's **raw TypeScript**:

```
Error [ERR_MODULE_NOT_FOUND]: Cannot find module '…/packages/i18n/src/app-config'
  imported from …/packages/i18n/src/main.ts
```

(Node's ESM loader can't run `.ts` and requires explicit file extensions — `import "./app-config"`
has neither.)

**Fix:** tell the Node-target build tool to **bundle** the workspace package into its output instead
of externalizing it, so its TypeScript is compiled at build time and Node never loads source:

```ts
// apps/desktop/electron.vite.config.ts
import { defineConfig, externalizeDepsPlugin } from "electron-vite";

export default defineConfig({
  main: {
    // Externalize node_modules (default) EXCEPT the workspace package — the main
    // process is Node and must not import its raw TS at runtime.
    plugins: [externalizeDepsPlugin({ exclude: ["@scope/i18n"] })],
    // …
  },
});
```

Verify the build output (e.g. `out/main/index.js`) contains **no** `@scope/i18n` import (grep it) —
the code is inlined.

**Rule of thumb:** a package consumed by a **bundler** target can stay source-only; a package
consumed by a **Node** target (a desktop main process, a Node script) must either be bundled by the
consumer (above) or resolved to built JS. Bundling is the simplest and needs no dist at dev time.

## Gotcha: match the package's `vitest` major to the app's

A shared package with its own `vitest` devDependency **must pin the same major** the consuming apps
use. Testing-library augmentation like `@testing-library/jest-dom/vitest` augments `vitest`'s
`Assertion` interface via `declare module "vitest"`. If two vitest majors land in the tree (e.g. a
package on `vitest@2` while the app is on `vitest@4`), the augmentation targets the wrong instance
and **every** `expect(...).toBeInTheDocument()` breaks — a TS2339 at type-check and
`Invalid Chai property` at runtime — across the app's otherwise-untouched tests.

Diagnosis tip: to tell a dep-induced failure from a real pre-existing one, check CI on `main`
(`gh run list`), **not** a local `git stash` — stashing reverts code but not `node_modules`, so a
locally-installed conflicting dep still poisons the baseline.

## Checklist for a new shared package

1. `tsdown.config.ts` with the entry list, `dts: true`, and `exports.devExports: true`
   (+ `customExports` for any non-TS assets like JSON).
2. `build` script = `tsdown`; run it once so `package.json` gets its `exports` / `publishConfig`.
3. `vitest` (if used) pinned to the **apps' major**.
4. If a **Node** runtime consumes it, exclude it from that consumer's externalization
   (e.g. electron-vite `externalizeDepsPlugin({ exclude })`), or point that consumer at built dist.
5. `dist/` stays git-ignored; `turbo run build` handles prod.
