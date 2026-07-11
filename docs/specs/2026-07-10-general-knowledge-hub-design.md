# Design: `general-knowledge` as a central knowledge hub

- **Date:** 2026-07-10
- **Status:** Approved (design), pilot phase pending implementation
- **Author:** Cristian Sotomayor (+ Claude)
- **Repo:** `csdev19/general-knowledge` (public)

## Problem

Valuable, hard-won documentation (DDD + hexagonal architecture, bounded contexts,
repository pattern, error handling, desktop/mobile/web/api stack setups, monorepo
and package build strategy) is scattered and duplicated across multiple projects
(`monorepo-template`, `rakoi-monorepo`, `kaipu-record-monorepo`, ...). Each new
project copy-pastes docs and drags along dead files that reference stacks the
project didn't choose (e.g. Elysia docs in a Hono-only project).

The goal: **one public hub of reusable knowledge**, so a fresh copy of the template
arrives clean and just *links* to the relevant, already-prepared knowledge instead
of carrying copies.

## Goals

1. `general-knowledge` becomes the single source of truth for reusable, stack-agnostic
   and stack-specific knowledge.
2. Knowledge is organized flat by topic AND composed into ready-to-use "stack recipes".
3. The template's `README` points to the hub (and to the specific stack recipe chosen
   via `bun run customize`) instead of shipping its own copies.
4. Product-specific content is **generalized** (product names removed) during migration.

## Non-goals

- Not building a docs website (Astro/Starlight). Plain Markdown, readable on GitHub.
- Not deleting/refactoring the template's `apps/documentation` in the pilot phase
  (that is a later phase).
- Not migrating product feature docs (e.g. `rakoi/wallet`, `kaipu/recording`) as
  product docs — only the reusable *patterns* extracted from them.

## Decisions (locked)

| Decision | Choice |
|---|---|
| Hub role | Central hub; template references it (README → `stacks/<chosen>.md`) |
| Structure | **Both**: flat knowledge by topic + a `stacks/` composition layer |
| Format | Plain Markdown (`.md`), no build |
| Migration style | **Generalize** — strip product names, keep the pattern |
| Rollout | **Pilot first**: `architecture` + `error-handling` + 1 stack recipe, validate, then the rest |
| `backlog/` folder | Documented as a convention in the hub; template ships an empty pre-scaffolded `backlog/` |

## Target structure

Two layers:

**Layer A — flat knowledge by topic (source of truth, reusable):**

```
general-knowledge/
├── README.md                     # master index: what exists + how to navigate
├── architecture/                 # DDD + hexagonal, bounded contexts, shared kernel,
│                                 #   repository pattern, domain modeling, ADRs,
│                                 #   observability, security hardening
├── error-handling/               # result types, error handlers, response helpers,
│                                 #   api-response-types, error-handling overhaul
├── web/                          # TanStack Start, routing, data-loading, server fns, web-ui
├── api/                          # hono + orpc, elysia + eden, auth, api-contract patterns
├── mobile/                       # Expo / React Native patterns
├── desktop/                      # Electron vs Tauri, IPC contract, main/renderer arch,
│                                 #   recording pipeline, permissions & onboarding
├── monorepos/                    # turborepo, bun workspaces, CI/CD pipelines, release-please
├── packages/                     # shared-package build strategy, infra-* naming, repo contracts
└── conventions/                  # schemas-first, constants/enums-as-const, backlog pattern,
                                  #   specs+plans workflow
```

**Layer B — `stacks/` composition (what the template links to):**

```
stacks/
├── README.md                     # table: stack → use case → link
├── fullstack-hono-orpc.md        # recipe: links into architecture/, error-handling/, web/, api/#hono, packages/
├── fullstack-elysia-eden.md      # recipe: architecture/, error-handling/, web/, api/#elysia
├── fullstack-convex.md           # recipe: architecture/, web/, realtime
├── mobile-expo.md                # recipe: mobile/, api/, architecture/
└── desktop-electron.md           # recipe: desktop/, architecture/, packages/
```

A `stacks/*.md` file does NOT duplicate content. It is a recipe:
- 1-paragraph "when to use this stack",
- a linked reading-list into Layer A docs (in order),
- 3–5 assembly notes specific to wiring that stack together.

## Source → topic mapping (discovered)

| Topic | Primary source | Key files |
|---|---|---|
| architecture | `kaipu` `architecture/` (clean) + `rakoi` (depth) | bounded-contexts-complete-guide, shared-kernel-*, repository-*, domain-architecture-patterns, domain-modeling-strategy, decisions/ADRs, observability, security-hardening |
| error-handling | `kaipu`/`rakoi` `backend/` | result-types, error-handlers, response-helpers, api-response-types + rakoi `observability/error-handling-overhaul` |
| web | `rakoi`/`kaipu` `frontend/` | data-loading, server-functions, web-ui-package |
| api | `kaipu` `stack/hono` + `rakoi` ADR | hono, orpc-hono-cloudflare-api-layer, api-contract, auth |
| mobile | `rakoi` `frontend/mobile-app` | mobile-app |
| desktop | `kaipu` `desktop/` (full set) | electron-vs-tauri, ipc-contract, main-process-architecture, renderer-architecture, recording-pipeline, permissions-and-onboarding, product-philosophy, filesystem-first-monetization |
| monorepos | `kaipu` `deployment/` + `testing/` + turbo | ci-cd-pipelines, pr-checks, release-automation-release-please |
| packages | `kaipu` `architecture/shared-package-build-strategy` + infra-naming | shared-package-build-strategy, infrastructure-naming, repository-contracts-and-implementations |
| conventions | both repos | constants-pattern, enums-as-const, schemas-implementation, backlog pattern, specs+plans workflow |

Absolute source roots:
- `/Users/cristiansotomayor/Documents/Workspace/Personal/Niway/kaipu-record-monorepo/apps/documentation/src/content/docs/`
- `/Users/cristiansotomayor/Documents/Workspace/Personal/Niway/rakoi-monorepo/apps/documentation/src/content/docs/`

## Migration rules

1. **Generalize**: remove product names (rakoi, kaipu, wallet, recording, etc.),
   replace with generic domain terms. Keep the pattern, the "why", and code shape.
2. **Deduplicate**: where both repos have the same doc, prefer the cleaner (kaipu)
   version and graft extra depth from rakoi.
3. **Convert `.mdx` → `.md`**: strip Starlight-specific frontmatter/components
   (`:::note`, `<Tabs>`, etc.) into plain Markdown equivalents.
4. **Preserve ADRs** as decisions but strip project-specific context.
5. Every topic folder gets a short `README.md` index (per the plain-markdown + index navigation).

## The `backlog/` convention

`kaipu` keeps a `backlog/` of future work items (roadmap, ideas, deferred specs).
The user relies on this heavily. Capture it as:
- `conventions/backlog-pattern.md` — what it is, how to run it, why it helps.
- Template ships an empty `backlog/` scaffold so new projects "come set up".
- Same treatment for the `specs/` + `plans/` (superpowers) workflow both repos use.

## Template ↔ hub wiring (later phase)

- `monorepo-template/README.md` gains a "Knowledge Documentation" section linking to
  the hub and to `stacks/<chosen>.md`.
- `bun run customize` prints/inserts the chosen stack's hub link on completion.
- `apps/documentation` is slimmed to template-operational docs only (how to run,
  customize, commands); transversal knowledge moves to the hub.

## Phased plan

- **Phase 0 — Scaffolding**: create hub folder structure + master `README.md` +
  `stacks/README.md` + topic `README` stubs.
- **Phase 1 — Pilot migration**: migrate & generalize `architecture/` and
  `error-handling/` fully; write one `stacks/fullstack-hono-orpc.md` recipe.
  **Validation checkpoint with user.**
- **Phase 2 — Remaining topics**: web, api, mobile, desktop, monorepos, packages,
  conventions (incl. backlog pattern).
- **Phase 3 — Remaining stack recipes**: elysia, convex, mobile-expo, desktop-electron.
- **Phase 4 — Template rewiring**: slim `apps/documentation`, wire README + customize
  to the hub, add empty `backlog/` scaffold.

## Success criteria

- A new template clone contains zero copied transversal knowledge docs; it links to
  the hub instead.
- Each stack recipe reads top-to-bottom as a "how to build with this stack" guide,
  composed entirely of links into flat topic docs + a few assembly notes.
- No product names leak into Layer A knowledge docs.

## Open questions / risks

- Exact stack recipes to seed depends on which patterns the template keeps after cleanup.
- Some rakoi docs use Effect (`effect-use-cases`, ADR-0001); decide whether Effect is
  a hub-level pattern or project-specific (defer to Phase 2).
