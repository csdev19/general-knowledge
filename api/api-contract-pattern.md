# API Contract Pattern

_Where the oRPC contract lives (in the server app, consumed via cross-app import) and how to extract it into a dedicated `packages/api-contract/` package._

## Current State

The oRPC contract lives inside `apps/server-hono/src/contract/` and the web app imports it via a cross-app relative path:

```
apps/web-hono/src/lib/orpc-client.ts
  → import { contract } from "../../../server-hono/src/contract"
```

This mirrors the Eden Treaty pattern (which imports `type { App }` from the server app). Keeping the contract in the server app is the simplest starting point: no extra package, no workspace wiring, and the web app still gets full type safety through the relative import.

## Why Extract to a Package

Extract once a second consumer needs the contract or the relative import becomes fragile:

- **Cleaner imports**: `@app/api-contract` instead of fragile relative paths
- **Multiple consumers**: Mobile app, CLI tools, or other services can depend on the contract
- **Explicit dependency graph**: Workspace deps are visible in `package.json`
- **Build isolation**: The build tool (Turborepo) can cache the contract independently

## Migration Steps

### 1. Create `packages/api-contract/`

```
packages/api-contract/
├── package.json
├── tsconfig.json
└── src/
    ├── index.ts
    └── modules/
        └── todo/
            └── todo.contract.ts
```

### 2. `packages/api-contract/package.json`

```json
{
  "name": "@app/api-contract",
  "version": "1.0.0",
  "type": "module",
  "exports": {
    ".": "./src/index.ts",
    "./todos": "./src/modules/todo/todo.contract.ts",
    "./package.json": "./package.json"
  },
  "dependencies": {
    "@app/domain": "workspace:*",
    "@orpc/contract": "catalog:",
    "zod": "catalog:"
  },
  "devDependencies": {
    "@app/config": "workspace:*",
    "typescript": "catalog:"
  }
}
```

### 3. `packages/api-contract/tsconfig.json`

```json
{
  "extends": "@app/config/tsconfig.base.json",
  "compilerOptions": {
    "composite": false,
    "outDir": "dist",
    "baseUrl": ".",
    "moduleResolution": "bundler"
  },
  "include": ["src/**/*"]
}
```

### 4. Move Contract Files

```bash
# Move from server app to package
cp apps/server-hono/src/contract/todo.contract.ts packages/api-contract/src/modules/todo/
cp apps/server-hono/src/contract/index.ts packages/api-contract/src/
```

No code changes needed — the files use `@app/domain/schemas` imports which work in both locations.

### 5. Update Server Imports

In `apps/server-hono/`:

```diff
# src/router.ts
-import { contract } from "./contract";
+import { contract } from "@app/api-contract";

# src/modules/todo/todo.router.ts
-import { todoContract } from "../../contract/todo.contract";
+import { todoContract } from "@app/api-contract/todos";
```

Add to `apps/server-hono/package.json` dependencies:

```json
"@app/api-contract": "workspace:*"
```

Remove `@orpc/contract` from the server app deps (it comes through the api-contract package).

### 6. Update Web Imports

In `apps/web-hono/src/lib/orpc-client.ts`:

```diff
-import { contract, type Contract } from "../../../server-hono/src/contract";
+import { contract, type Contract } from "@app/api-contract";
```

Add to `apps/web-hono/package.json` dependencies:

```json
"@app/api-contract": "workspace:*"
```

### 7. Clean Up

```bash
# Remove old contract from server app
rm -rf apps/server-hono/src/contract/

# Install
bun install

# Verify
bun run check-types
```

## Architecture After Migration

```
packages/api-contract/     Contract definitions (schemas, routes, types)
  ↑                    ↑
server-hono            web-hono
(implements via        (consumes via
 @orpc/server)          OpenAPILink)
```

Both apps depend on `@app/api-contract` as a workspace dependency. The contract imports domain schemas from `@app/domain/schemas`, keeping it aligned with the existing layer architecture.
