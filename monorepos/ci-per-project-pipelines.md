# Per-project CI pipelines

_Split the monolithic PR check into a fast universal compile check plus path-filtered per-project test workflows, skip release PRs, and normalize release-* naming._

## Problem

CI started as one monolithic workflow, `pr-validation.yml` ("Validate PR"), triggered on **every**
PR to `main` with no path filter. It ran lint + format + build-all + type-check-all + build-web +
build-backend + type-check-desktop **and the full desktop test suite** in a single job. Two
consequences:

- **No scoping.** A web-only or api-only PR still ran the desktop tests, and vice versa. Everything
  ran on everything.
- **Release PRs paid the full price.** release-please chore PRs (version + CHANGELOG only) touch
  `apps/<app>/package.json`, so they matched the desktop E2E path filter and always ran
  `pr-validation` — running the heavy suites on mechanical version bumps.

The dominant cost is the desktop unit suite: **~2m 9s in CI** (the same 600 tests run in ~10s
locally — a CI-parallelism gap tracked as a follow-up, not fixed here).

## Goal

- A **fast universal check** on every PR that answers "does everything still compile, lint and
  format?" — the always-on green light.
- **Per-project test suites** that run **only when that project changes** (or a shared package it
  depends on) — never on unrelated PRs.
- **Zero ceremony on release PRs.**
- Consistent **`release-*`** naming for all release/deploy workflows.

## Design

### Universal check — `ci.yml` ("CI")

Runs on every PR (skips release-please branches via `if`). One job:
lint → format → build `@<scope>/*` → type-check all → build web → build backend → type-check
desktop. **No tests.** ~1–1.5 min. This is `pr-validation` minus the desktop test step.

> **Do not add `paths-ignore` for markdown here.** See
> [Docs-only PRs still need the formatter](#docs-only-prs-still-need-the-formatter) — skipping the
> universal check on docs lets malformed markdown land unchecked.

### Per-project tests (path-filtered, skip release PRs)

| Workflow         | Name              | Paths                                             | Jobs                                                        |
| ---------------- | ----------------- | ------------------------------------------------- | ----------------------------------------------------------- |
| `ci-desktop.yml` | **Desktop Tests** | `apps/desktop/**`, `packages/**`, `bun.lock`, self| `unit` (vitest) + `e2e` (Electron/Playwright), in parallel  |
| `ci-web.yml`     | **Web Tests**     | `apps/web/**`, `packages/**`, `bun.lock`, self    | `test` — runs the web app's `test` script when present      |
| `ci-api.yml`     | **API Tests**     | `apps/api/**`, `packages/**`, `bun.lock`, self    | `test` — runs the api app's `test` script when present      |

Each test job carries `if: ${{ !startsWith(github.head_ref, 'release-please--') }}` so release PRs —
which match the path filter via `package.json` — run nothing.

`packages/**` is in all three filters on purpose: a shared-package change can break any consumer, so
all three test suites should run.

**Desktop unit job needs no package build:** if the desktop app's only workspace dep is resolved
from source (e.g. tsdown devExports), vitest runs directly after install (verify locally).

**Web/api can be wired but test-less:** if neither has a `test` script yet, the job's step checks
for one and no-ops with a note until it's added (building `@<scope>/*` first only when there is
actually a test to run). Adding a `test` script is all it takes to turn each on.

### Naming normalization

If desktop already used `release-desktop.yml` ("Release Desktop") while web/api used `deploy-web.yml`
/ `deploy-api.yml`, rename to **`release-web.yml`** ("Release Web") and **`release-api.yml`**
("Release API") so every tag-triggered publish workflow reads `release-*`. Triggers
(`web-v*` / `api-v*`) are unchanged.

## What runs when

| A PR that touches…    | CI (compile) | Desktop Tests | Web Tests | API Tests |
| --------------------- | ------------ | ------------- | --------- | --------- |
| only desktop          | ✅           | ✅            | —         | —         |
| only web              | ✅           | —             | ✅        | —         |
| only api              | ✅           | —             | —         | ✅        |
| a shared `packages/*` | ✅           | ✅            | ✅        | ✅        |
| only docs / `*.md`    | ✅           | —             | —         | —         |
| a release-please PR   | —            | —             | —         | —         |

The docs row is the one worth reading twice: the compile check **does** run, the test suites do
not. See below.

## Docs-only PRs still need the formatter

The tempting simplification is to `paths-ignore: "**/*.md"` on the universal check too, so a
documentation PR runs nothing at all. **Don't.** If the repo's formatter covers markdown — `oxfmt`
and Prettier both do, including `.mdx` — then the universal check is the only thing standing
between a malformed table and `main`.

What that looks like when you skip it: a docs PR merges green, and the next unrelated code PR fails
on `format --check` complaining about a file its author never touched. The failure surfaces far
from its cause, in someone else's branch. Rebases make it worse — resolving a conflict inside a
markdown table routinely leaves the column padding misaligned, which the formatter rejects and a
human reviewer never notices.

So the split is by **cost**, not by file type:

| Half           | Contents                       | Runs on             | Why                                                              |
| -------------- | ------------------------------ | ------------------- | ---------------------------------------------------------------- |
| Fast gate      | lint, format check, type-check | **every** PR        | Seconds. Covers markdown. Nothing here is worth skipping.        |
| Slow half      | test suites, builds            | code PRs only       | Minutes. A changelog edit cannot break a test.                   |

Two workflows is enough for a small monorepo — a `pr-validation.yml` holding the fast gate and a
`pr-tests.yml` holding the rest, the latter carrying the `paths-ignore`. Scale up to the
per-project split above only once individual suites get slow enough to be worth isolating.

### `paths-ignore` semantics worth knowing

A run is skipped only when **every** changed file matches an ignore pattern. A PR touching both
code and documentation still gets the full suite — which is what you want, and means the filter
needs no escape hatch for mixed PRs.

Ignore the docs app's directory (`apps/docs/**`, `apps/documentation/**`) alongside `**/*.md` and
`**/*.mdx`, but confirm first that nothing under it is actually tested or built: check whether it
has a `test` script and whether its package name is inside the scope your build filters on
(`--filter='@<scope>/*'`). If the docs app is outside that scope, ignoring it costs nothing.

### Renaming jobs breaks required checks

Splitting one workflow into two renames the status checks. If branch protection requires the old
name, every PR strands on "Expected — waiting for status" until the setting is updated. Check the
protection rules **before** merging the split, not after. (On a private repo without Pro, branch
protection is unavailable and checks are advisory — nothing to update, but worth re-verifying if
the repo ever goes public or upgrades.)

## Files

- Add: `.github/workflows/ci.yml`, `ci-desktop.yml`, `ci-web.yml`, `ci-api.yml`.
- Remove: `.github/workflows/pr-validation.yml`, `e2e-desktop.yml` (superseded).
- Rename: `deploy-web.yml` → `release-web.yml`, `deploy-api.yml` → `release-api.yml`.
- Unchanged: `release-please.yml`, `release-desktop.yml`.

If no branch protection exists (e.g. a private repo without Pro), checks are advisory and skipping
CI on release PRs never blocks a merge.

## Follow-ups (out of scope)

- **CI test speed:** the desktop unit suite is ~10s locally but ~2m 9s in CI — likely vitest
  environment setup not parallelizing on the 2-core runner. Worth investigating (sharding / lighter
  test environment); would make even the per-project suite fast.
- **Web/api test suites:** add a `test` script to each; the workflows then activate.
