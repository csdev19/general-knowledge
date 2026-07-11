# Monorepo structure & tooling

_The layout and tooling of a Turborepo + Bun-workspaces monorepo — workspaces, the shared dependency catalog, Turbo task graph, and lint/format/git-hook setup._

## Layout

```
apps/*        Deployable applications (web, api, desktop, documentation, …)
packages/*    Shared libraries consumed by apps, layered by architecture:
              packages/domain/        Pure. Constants, schemas, types, interfaces.
              packages/application/    Use cases. Depend only on domain interfaces.
              packages/infra-db/       Drizzle repos, mappers, schemas (server-only).
              packages/infra-auth/     Better Auth configuration (server-only).
              packages/web-ui/         Shared UI components (built to dist/).
```

Package manager: **Bun** (`packageManager: "bun@1.3.4"`). Build orchestration: **Turborepo**.

## Bun workspaces + catalog

`package.json` at the repo root declares the workspaces and a shared **dependency catalog**:

```json
{
  "workspaces": {
    "packages": ["apps/*", "packages/*"],
    "catalog": {
      "zod": "4.3.6",
      "hono": "4.12.3",
      "@orpc/contract": "1.13.5",
      "@orpc/server": "1.13.5",
      "better-auth": "1.4.18",
      "tailwindcss": "4.1.18",
      "typescript": "5.9.3"
    }
  }
}
```

- `workspaces.packages` globs `apps/*` and `packages/*` into one install.
- The **`catalog`** pins shared dependency versions in one place. Individual packages reference them
  with `"zod": "catalog:"` instead of a hardcoded version, so every workspace resolves the same copy
  and version bumps happen once at the root. (See `dependencies` / `devDependencies` in leaf packages
  using `catalog:`.)

## Turborepo task graph

`turbo.json` defines the task pipeline. Key ideas:

```json
{
  "ui": "tui",
  "concurrency": "20",
  "tasks": {
    "transit":     { "dependsOn": ["^transit"] },
    "build":       { "dependsOn": ["^build"], "inputs": ["$TURBO_DEFAULT$", ".env*"], "outputs": ["dist/**"] },
    "lint":        { "dependsOn": ["transit"] },
    "check-types": { "dependsOn": ["transit"] },
    "dev":         { "cache": false, "persistent": true },
    "db:push":     { "cache": false, "env": ["DATABASE_URL"] },
    "db:studio":   { "cache": false, "persistent": true, "env": ["DATABASE_URL"] },
    "db:migrate":  { "cache": false, "env": ["DATABASE_URL"] },
    "db:generate": { "cache": false, "env": ["DATABASE_URL"] }
  }
}
```

- **`^` prefix** (`^build`, `^transit`) means "run this task in all _dependencies_ first" — the
  topological build order across the workspace graph.
- **`transit`** is a lightweight "prepare source exports" task that `lint` and `check-types` depend
  on, so type-aware tasks see freshly generated declaration/export files first.
- **`build`** caches on `outputs: ["dist/**"]` and includes `.env*` in its `inputs` so an env change
  invalidates the cache.
- **`dev`** is `persistent: true` + `cache: false` (long-running watch server, never cached).
- **`db:*`** tasks are uncached and declare `env: ["DATABASE_URL"]` so Turbo knows they depend on
  that variable.

## Root scripts

```jsonc
{
  "dev":          "turbo run dev",
  "build":        "turbo run build",
  "build:no-cache":"turbo run build --force",
  "check-types":  "turbo run check-types",
  "lint":         "oxlint",
  "format":       "oxfmt --write .",
  "check":        "bun run lint && bun run format",
  // DB commands load env via dotenvx, then run a Turbo task filtered (-F) to the infra-db package:
  "db:push":      "dotenvx run -f apps/api/.env -- turbo run db:push -F @<scope>/infra-db",
  "db:studio":    "dotenvx run -f apps/api/.env -- turbo run db:studio -F @<scope>/infra-db"
}
```

- `turbo run <task> -F <pkg>` filters a task to a single workspace.
- DB scripts wrap Turbo in **`dotenvx run -f apps/api/.env`** so the DB connection env is loaded from
  the api app's `.env` before the task runs. Run these from the **repo root**, not from inside the
  package.

## Lint / format / git hooks

- **Linting:** [`oxlint`](https://oxc.rs) (`bun run lint`).
- **Formatting:** [`oxfmt`](https://oxc.rs) (`bun run format`, or `format:tracked` to format only
  git-tracked source files).
- **Git hooks:** [`husky`](https://typicode.github.io/husky/) (`prepare: "husky"`) +
  [`lint-staged`](https://github.com/lint-staged/lint-staged) runs `oxlint` + `oxfmt --write` on
  staged JS/TS files and `oxfmt --write` on staged JSON/CSS on every commit:

```json
"lint-staged": {
  "*.{js,jsx,cjs,mjs,ts,tsx,cts,mts}": ["oxlint", "oxfmt --write"],
  "*.{json,jsonc,css}": ["oxfmt --write"]
}
```

## Notes

- `"type": "module"` at the root — the whole repo is ESM.
- A root `bunfig.toml` configures Bun's install behavior. This repo uses the **isolated linker**:

  ```toml
  [install]
  linker = "isolated"
  ```

  The isolated linker gives each package a strict `node_modules` containing only its declared
  dependencies (pnpm-style), preventing "phantom dependency" bugs where a package imports something
  it never declared but that happened to be hoisted to the root.
