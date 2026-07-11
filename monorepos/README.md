# Monorepos

How to build and operate a **Turborepo + Bun-workspaces** monorepo: repository structure and
tooling, CI/CD per project, PR checks, release automation with release-please, and a layered testing
strategy across the workspace.

These docs are generalized patterns — product names and app-specific worker names have been replaced
with generic placeholders (`<app>`, `<scope>`, `apps/web`, `apps/api`, `apps/desktop`). The
patterns, rationale, pipeline shapes, and commands are preserved.

## Contents

| File                                                       | Summary                                                                     |
| ---------------------------------------------------------- | --------------------------------------------------------------------------- |
| [monorepo-structure.md](./monorepo-structure.md)           | Workspace layout, Bun catalog, Turbo task graph, lint/format/hooks tooling. |
| [ci-cd-pipelines.md](./ci-cd-pipelines.md)                 | Tag-driven independent release/deploy pipelines to Cloudflare Workers + R2. |
| [pr-checks.md](./pr-checks.md)                             | Two PR-gate workflows (validation + desktop E2E) and why they stay split.   |
| [ci-per-project-pipelines.md](./ci-per-project-pipelines.md)| Universal compile check plus path-filtered per-project test suites.         |
| [release-automation.md](./release-automation.md)           | release-please manifest mode for drift-free, per-app versioning.            |
| [testing-strategy.md](./testing-strategy.md)               | Layered testing model (unit → component → integration → E2E) overview.      |

### testing/

| File                                                              | Summary                                                                |
| ----------------------------------------------------------------- | ---------------------------------------------------------------------- |
| [testing/unit-and-component.md](./testing/unit-and-component.md)   | Co-located Vitest unit/component tests, fixtures, selector priority.   |
| [testing/integration.md](./testing/integration.md)                | In-process API tests against a real Neon branch (no mocks, no server). |
| [testing/e2e.md](./testing/e2e.md)                                | Playwright full-stack E2E: hydration, auth reuse, testid selectors.    |
| [testing/ci.md](./testing/ci.md)                                  | Running the suites in CI on a shared Neon branch; drift/quoting traps. |
