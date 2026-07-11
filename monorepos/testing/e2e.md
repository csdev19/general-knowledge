# End-to-End Tests

_Playwright drives full-stack tests in a real browser against a dedicated Neon branch, booting both apps as web servers — with rules for hydration, auth reuse, and testid-first selectors._

Playwright drives full-stack tests in a real browser against a dedicated Neon branch. The Playwright
config boots both apps as web servers: the api via `wrangler dev` (port 3101) and the web app via
`vite dev` (port 3100). They read env from `apps/api/.env` and `apps/web/.env`.

## Running

```bash
bun run test:e2e                       # headless
bun run --cwd apps/web test:e2e:ui     # interactive UI mode
```

## DB strategy

- One Neon branch, shared with the integration suite.
- Not wiped between tests. Each test creates uniquely-named data via `uniqueEmail()` /
  `uniqueName()` (`apps/web/e2e/fixtures/unique.ts`) and asserts only on that data — never on counts
  or totals, which accumulate.
- `bun run test:clean` truncates the branch if it gets noisy.

## Auth reuse

`auth.setup.ts` signs up one fresh user, stores the session in `e2e/.auth/user.json`, and the
authenticated `chromium` project reuses it via `storageState`. `storageState` is set on the
`chromium` project, **not** in the global `use` — otherwise the setup project tries to read the file
before it exists (fails on a clean checkout). Specs that must be signed out (`auth-flows.spec.ts`)
opt out with `test.use({ storageState: { cookies: [], origins: [] } })`.

## Wait for hydration before interacting

The web app server-renders, then React hydrates and attaches form handlers. If a test fills and
submits during that gap, the browser does a **native** form submit instead of calling the API, so
nothing happens. Always navigate with the helper:

```ts
import { gotoHydrated } from "./fixtures/hydrate";
await gotoHydrated(page, "/groups");
```

`gotoHydrated` waits for `body[data-hydrated]`, a flag the root layout sets in a `useEffect` after
mount. Use it for every full navigation. This is the single most important reliability rule for
these specs — without it they pass locally (warm) and fail in CI (cold).

## Selectors: testid-first for forms

Form and action elements use `data-testid` hooks (e.g. `create-group-trigger`, `group-name-input`,
`add-expense`, `entry-amount`, `bill-submit`, `settle-confirm`). This keeps specs independent of
i18n copy — labels render in the user's locale, testids do not. For static text created in-test (a
group or bill name), assert with `getByText(uniqueName)`.

When a list has many rows, scope by the unique text then act on the testid inside:

```ts
const row = page.getByTestId("entry-row").filter({ hasText: desc });
await row.getByTestId("entry-edit").click();
```

## Writing new specs

- Place under `apps/web/e2e/<flow>.spec.ts`.
- `gotoHydrated` for navigation; testids for interaction; `getByText(uniqueName)` for assertions.
- Data-assertions only — no pixel snapshots.
- Don't assume an empty state; assert on data you created in-test.

## How the browser reaches the API

The browser talks only to the web app (3100). `/api/*` is proxied to the api (3101). In production
this uses a Cloudflare Service Binding; locally/CI there is no binding, so the proxy falls back to
plain HTTP at `VITE_SERVER_URL` (3101). That env var must be set for the web server, or the proxy
targets the binding placeholder and every API call fails.

## Config pins

- Web port: `E2E_WEB_PORT` (default 3100); API port: `E2E_API_PORT` (default 3101).
- The api `dev:test` script (`wrangler dev`, no hardcoded port) is what Playwright runs, so the
  `--port` it appends is the only one — the normal `dev` script hardcodes 3000 and would clash.
- `BETTER_AUTH_URL` must match the API port so Better Auth cookies are scoped correctly.
