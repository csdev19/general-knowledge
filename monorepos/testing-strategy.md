# Testing Strategy

_A layered testing strategy for a full-stack monorepo — each layer catches a different class of bug; together they cover pure logic, real backend behavior, and full-stack user flows._

Use a layered testing strategy. Each layer catches a different class of bug; together they cover
pure logic, real backend behavior, and full-stack user flows.

| Layer             | Runner               | DB        | Scope                                                  |
| ----------------- | -------------------- | --------- | ------------------------------------------------------ |
| Domain unit       | Vitest (node)        | none      | Schemas, constants, pure helpers                       |
| Application unit  | Vitest (node)        | mocked    | Use cases with mocked repositories                     |
| Component / route | Vitest + jsdom + RTL | mocked    | Web components and route components                    |
| API integration   | Vitest (node)        | real Neon | API (Hono) routes in-process against a real branch     |
| E2E               | Playwright           | real Neon | Full-stack golden paths in a real browser              |

Integration tests for the DB infrastructure package (`@<scope>/infra-db`) are intentionally out of
scope. You rely on Neon + Drizzle, and exercise the DB through the API integration layer instead.

## What each layer is for

- **Unit / component** — fast, no I/O. Run on every change. Prove logic and rendering in isolation.
  See [Unit and Component Tests](./testing/unit-and-component.md).
- **API integration** — call the real API app against a real Neon branch (no mocks). Prove the API
  contract, auth, and DB queries actually work end to end. See
  [API Integration Tests](./testing/integration.md).
- **E2E** — drive a real browser through the proxy, API, and DB. Prove a user can complete a flow
  and catch wiring/hydration bugs the lower layers can't. See [End-to-End Tests](./testing/e2e.md).

There is intentional overlap (e.g. "create a group" exists at both the API and E2E level) — each
layer fails for different reasons, so the overlap is coverage, not waste.

## Common commands

```bash
bun run test              # all unit + component (no DB), parallel
bun run test:unit         # domain + application only
bun run test:web          # web component/route tests
bun run test:integration  # API integration (real Neon branch)
bun run test:e2e          # Playwright E2E (boots api + web)
bun run test:clean        # truncate the Neon test branch
```

`bun run test` does **not** touch the database. The integration and E2E suites do — they need a Neon
test branch configured (below).

## Environment

- **Local integration tests**: `apps/api/.env.test` (copy from `.env.test.example`, gitignored). The
  `test:integration` script loads it via dotenvx; `DATABASE_URL` defaults to `${TEST_DATABASE_URL}`
  so it reuses the test branch URL already in `apps/web/.env`.
- **Local E2E**: the Playwright config boots the api (`wrangler dev`) and web (`vite dev`), which
  read `apps/api/.env` and `apps/web/.env`.
- **CI**: a single shared Neon test branch, injected from the `TEST_DATABASE_URL` GitHub secret. See
  [Continuous Integration](./testing/ci.md).

## Layer details

- [Unit and Component Tests](./testing/unit-and-component.md)
- [API Integration Tests](./testing/integration.md)
- [End-to-End Tests](./testing/e2e.md)
- [Continuous Integration](./testing/ci.md)
