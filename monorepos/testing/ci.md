# Continuous Integration (testing)

_How the test suites run in CI against a single shared Neon test branch — two parallel jobs, one secret, and the schema-drift and quoted-URL gotchas. Which tier runs them is set by release-gated-verification.md._

**Which tier runs these jobs is decided in
[release-gated-verification.md](../release-gated-verification.md), not here.** This page used
to say CI runs on every PR to `main`; that is no longer the default. On a private
single-owner repository the pre-push hook runs `verify` and the cloud runs it on the ready
release candidate and at the tag. Every-PR checks are for repositories with several mergers
or public ones, and even then only on non-draft PRs
([pr-checks.md](../pr-checks.md#draft--ready-for-review-as-the-trigger)).

Wherever they run, the jobs below share one shape: they run in parallel, both bound to the
`testing` GitHub environment.

## Jobs

**`validate`** — lint, format check, build packages, type check, unit + component tests, then push
schema to the test branch, run the API integration suite, and build all apps.

**`e2e`** — install, build packages, push schema, write `.env` files for both dev servers, install
Chromium, run Playwright, upload the report artifact.

A weekly job (`.github/workflows/reset-test-db.yml`, Mondays + manual) truncates and re-syncs the
shared branch.

## Shared test branch

There is one permanent Neon branch for tests (not an ephemeral branch per PR). Both jobs push the
PR's schema to it and run against it. Tests isolate themselves with unique data, so parallel runs
don't collide. This keeps CI fast and needs only one secret.

## Secrets

The `testing` environment holds exactly one secret:

- `TEST_DATABASE_URL` — the Neon test branch **pooled** connection string.

Everything else the workflow needs (test auth secret, email/analytics placeholders) is a hardcoded
non-secret value in the workflow.

### Store the URL with no quotes

`TEST_DATABASE_URL` must be the raw string `postgresql://user:pass@host/db?...` with **no
surrounding quotes, whitespace, or trailing newline**. The neon driver runs `new URL()` on it; a
leading quote yields `ERR_INVALID_URL` and `db:push` fails. (A `.env` file tolerates quotes because
dotenv strips them — a raw CI env var does not.)

## Two `db:push` runs are intentional

Both jobs run `db:push:ci` because they execute on separate runners and each needs the schema present
before its tests. They target the same branch, so the second push is an idempotent no-op. Serializing
the jobs to push once would cost more wall-clock time than it saves.

## Gotcha: schema drift on the shared branch

`drizzle-kit push` is interactive for ambiguous diffs (it asks "is column X new or renamed from Y?").
In CI there is no TTY, so when the branch has drifted, push applies nothing for the ambiguous table
yet still exits 0 — the step shows green but the schema is stale, and queries fail at runtime with
`column ... does not exist`.

To recover, reset the branch to a clean schema so push has no ambiguity: drop the `<app>_*` tables
and re-push (a clean DB creates everything from scratch with no prompts). The weekly reset is the
durable place to do this.

## Debugging a failing run

- API errors are logged as `InternalError` and hide the cause. To see the real DB error temporarily,
  log `e.cause` in the failing use case's `catch` (NeonDbError carries `message` + `code`, e.g.
  `42703` = undefined column).
- The Playwright report artifact contains traces (network + DOM snapshots) for every failed spec.
- `gh run view <id> --log` streams job logs; the `[WebServer]` prefix is the booted dev servers'
  output during E2E.
