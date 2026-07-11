# API Integration Tests

_In-process tests that call the real API (Hono) app against a real Neon branch — no mocks, no HTTP server — proving the oRPC contract, auth, and Drizzle queries work end to end._

In-process tests that call the real API app against a real Neon branch — no mocks, no HTTP server.
They prove the oRPC contract, Better Auth, and Drizzle queries work end to end.

## Where they live

`apps/api/src/test/`:

- `helpers.ts` — `req()` (fires a request straight into the Hono app via `app.fetch`),
  `signUpAndGetCookie()`, `signInAndGetCookie()`, `authed()`, `uniqueEmail()`, `uniqueName()`.
- `support.ts` — typed `api()` wrapper plus entity builders (`createGroup`, `getDetail`,
  `addOfflineMember`, `getBalances`).
- `*.test.ts` — one file per module (auth, groups, expenses, settlements, friends, wallet, health).
- `cloudflare-workers-stub.ts` — see below.

Unit tests for the api app stay in `src/**/__tests__/` and mock their dependencies. Integration
tests are a separate Vitest project.

## How it works

The Hono app is the production entry (`src/index.ts`), exported as a Cloudflare Worker default. Tests
call it directly:

```ts
const res = await req("/api/v1/health");
```

No server is started, so tests are fast and deterministic. The `env` (DATABASE_URL, auth secret,
etc.) is read from `process.env`, populated from `.env.test` before the app imports.

### The `cloudflare:workers` stub

`src/env.ts` does `import { env } from "cloudflare:workers"` at module load, which only exists in the
workerd runtime and would crash under Node/Vitest. The integration Vitest config aliases
`cloudflare:workers` to `src/test/cloudflare-workers-stub.ts`, which re-exports `process.env`. So the
same app code runs under Node with env coming from the process.

## Running

```bash
bun run test:integration       # local — loads .env.test via dotenvx
bun run test:integration:ci    # CI — DATABASE_URL injected as a real env var
```

`vitest.integration.config.ts` defines the integration project. It runs serially
(`fileParallelism: false`) to avoid unique-constraint collisions when multiple workers create users
on the same branch.

## Setup

```bash
cp apps/api/.env.test.example apps/api/.env.test
```

`DATABASE_URL` defaults to `${TEST_DATABASE_URL}`, expanded from `apps/web/.env` by dotenvx — so if
you already have a test branch URL there, no edits are needed. The other values (auth secret, email
provider) are dummy test placeholders.

## Conventions

- One fresh user per `it()` via `uniqueEmail()` — the branch is never wiped between tests, so never
  share users.
- Assert on real outputs and status codes, not mock interactions.
- Response shape is `{ data: T | null, error: { message } | null }`.
- oRPC status codes: validation errors are `400`, `FORBIDDEN` is `403`, `NOT_FOUND` is `404`.
- A payment recorded by the creditor is auto-confirmed — re-confirming returns `400`.

## Gotcha: quoted connection strings

The neon driver rejects a `DATABASE_URL` with surrounding quotes or whitespace. dotenv-style loaders
strip quotes, but a raw env var (CI) does not — store the Neon URL with no quotes. See
[Continuous Integration](./ci.md).
