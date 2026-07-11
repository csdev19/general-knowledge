# Unit and Component Tests

_Fast, no-I/O tests for domain/application logic and web components, with co-located files, builder fixtures, and a selector priority for component tests._

## Where tests live

- Domain: `packages/domain/src/**/__tests__/*.test.ts`
- Application: `packages/application/src/**/__tests__/*.test.ts`
- Web components/routes: `apps/web/src/**/__tests__/*.test.tsx`

Co-located, not in a separate `tests/` tree.

## Fixtures

Builders: `@<scope>/domain/testing` exposes entity builders (e.g. `makeGroup`, `makeMember`,
`makeExpense`, `makeSettlement`, `makeMemberBalance`). Each accepts a `Partial<T>` override.

Repository mocks: `packages/application/src/__tests__/mocks/repos.ts` provides mock factories (e.g.
`mockGroupRepo()`, `mockMemberRepo()`, `mockExpenseRepo()`, `mockSettlementRepo()`,
`mockBalanceRepo()`, `mockInviteRepo()`). Each method is a `vi.fn()`.

Server functions: mock per test file with `vi.mock("@/server-functions/<name>")`. Keep typed shapes
identical to the real exports.

## Selector priority for component tests

1. `getByRole` with accessible name — reflects what users see.
2. `getByLabelText` for form inputs.
3. `getByText` only when roles/labels don't cover it.
4. `data-testid` as a last resort.

## React + web-ui dedup

The `vitest.config.ts` in the web app sets `resolve.conditions: ["development"]` and
`dedupe: ["react", "react-dom", "@base-ui/react"]` so tests use the workspace React copy rather than
the bundled `dist/` copy of the shared UI package (`@<scope>/web-ui`). Without this, `@base-ui/react`
hits "Invalid hook call" because two React instances coexist.
