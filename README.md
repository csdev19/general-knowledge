# General Knowledge

Central hub of reusable knowledge for building products: architecture, stacks
(web / api / mobile / desktop), error handling, monorepos and packages.

The idea: **one single source of truth**. New templates and projects **link here**
instead of dragging copies of the documentation around. Once you pick a stack,
everything is already prepared — you just consume it.

## How it is organized

Two layers:

- **Flat knowledge by topic** — the source of truth, reusable and (mostly)
  stack-agnostic. Each folder has its own `README.md` index.
- **[`stacks/`](./stacks/)** — recipes that **compose** the flat topics into a
  "how to build with this stack" guide. They do not duplicate content: they link
  and add the assembly notes specific to each stack. **This is what a template's
  README points at.**

## Topics

| Topic | What it contains |
| --- | --- |
| [architecture/](./architecture/) | DDD + hexagonal, bounded contexts, shared kernel, repository pattern, domain modeling, observability, security hardening, ADRs |
| [effect/](./effect/) | Effect as a default backend tool: why adopt it and when not to, adoption as a decision about **reach** (and the one capability that forces the boundary to move), the lazy-service pattern, and the traps — dishonest error channels, cached failures, runtime ownership, guards that pass for the wrong reason |
| [error-handling/](./error-handling/) | Result types, response helpers, api-response-types, error handlers, a centralization retrospective |
| [web/](./web/) | TanStack Start/Router/Query, data loading, server functions, shared UI package, bundle splitting |
| [api/](./api/) | Hono + oRPC on Cloudflare Workers, the api-contract pattern, isomorphic client, ADR, auth gotchas, **centralized auth service** (cross-origin topology: the CORS/`trustedOrigins`/cookies triad, KV as the session cache) |
| [convex/](./convex/) | Convex as a reactive backend: client connection (the two URLs, reactive `useQuery`, generated API in a monorepo) and **Better Auth hosted inside Convex** (SDK 57 versions, the hanging `useSession` fix, `expo-network`, overrides, `exp://`) |
| [mobile/](./mobile/) | Expo / React Native: structure + shared domain, **dev builds & Metro** (when to rebuild, emulator connection), **build & install** (debug/release × profile, `run:*` vs EAS, one install per environment, where env vars actually travel, **+ a copy-paste kit with the full scripts**) and **Google Maps** (dev build, key, Maps SDK Android + billing, SHA-1) |
| [desktop/](./desktop/) | Electron: main/renderer/preload, typed IPC, native permissions, local-first vault, media pipeline, distribution |
| [infra/](./infra/) | Domains, DNS and edge routing: **migrating a domain to Cloudflare Workers** — the inherited parking records that break the deploy, apex vs `www`, and the checklist of everything a new origin invalidates (auth, OAuth, CORS, deep links) |
| [monorepos/](./monorepos/) | Turborepo + Bun workspaces, CI/CD per project, PR checks, release-please, testing strategy |
| [packages/](./packages/) | The `infra-*` convention, build strategy (src vs dist), repository contracts, a package case study |
| [conventions/](./conventions/) | **English only**, constants/enums-as-const, schemas-first, **backlog pattern**, specs+plans workflow, **agent delegation** |

## Stacks (ready-made recipes)

See **[stacks/](./stacks/)** for the assembly guides:

| Stack | Use case |
| --- | --- |
| [fullstack-hono-orpc](./stacks/fullstack-hono-orpc.md) | Type-safe web + API (Hono + oRPC) on Cloudflare Workers. **Default.** |
| [fullstack-elysia-eden](./stacks/fullstack-elysia-eden.md) | Type-safe web + API (Elysia + Eden Treaty) |
| [fullstack-convex](./stacks/fullstack-convex.md) | Reactive fullstack with Convex (realtime) |
| [service-only-hono](./stacks/service-only-hono.md) | Backend service with no client, shared by several products (centralized auth) |
| [mobile-expo](./stacks/mobile-expo.md) | Expo mobile app on top of the shared domain + API |
| [desktop-electron](./stacks/desktop-electron.md) | Local-first Electron desktop app |

## How a template / project uses it

1. The project's `README` links to the chosen stack: `stacks/<stack>.md`.
2. The recipe walks it, in order, through the flat knowledge docs it needs.
3. No copy-pasting docs between projects — you link to this hub.

## Conventions worth adopting from day 1

- **[English only](./conventions/english-only.md)** — everything written into a repo is in
  English: docs, code, comments, commits, PRs. The only exception is localized product copy.
- **[Backlog pattern](./conventions/backlog-pattern.md)** — a `backlog/` folder with the map
  of "what's coming"; you never lose deferred work.
- **[Changelog pattern](./conventions/changelog-pattern.md)** — a decision journal in the
  docs app: one file per entry, auto-generated index (zero merge conflicts), complementing
  release-please's `CHANGELOG.md`.
- **[Specs + plans workflow](./conventions/specs-and-plans-workflow.md)** — brainstorm →
  spec (design) → plan (implementation) → archived on ship.
- **[Plan → backlog](./conventions/plan-to-backlog.md)** — turning an approved plan into
  self-sufficient backlog deliverables that agents execute in parallel.
- **[Release automation](./monorepos/release-please-playbook.md)** — release-please from day 1:
  `main` is integration, production is a tag, no `production` branch. Includes the closure rule
  that decides how many version lines a repo may have.
- **[Agent delegation](./conventions/ai-agent-delegation.md)** — the two modes
  (fast/expensive vs slow/cheap), which roles never get cheaper, and the measured evidence
  that what makes delegated work slow is the structure of the process, not the model tier.

---

_The design of this hub lives in [`docs/specs/2026-07-10-general-knowledge-hub-design.md`](./docs/specs/2026-07-10-general-knowledge-hub-design.md)._
