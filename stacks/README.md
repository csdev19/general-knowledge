# Stacks

Assembly recipes. Each file describes **how to build with one concrete stack**
and **links** to the flat knowledge docs (`../architecture/`, `../web/`, `../api/`, …)
that compose it. They do not duplicate content: they are an ordered reading list + the
assembly notes specific to that stack.

**A template/project README points at its stack's recipe.**

| Stack | Web | API | Realtime | Use case |
| --- | --- | --- | --- | --- |
| [fullstack-hono-orpc](./fullstack-hono-orpc.md) | TanStack Start | Hono + oRPC | — | Type-safe fullstack on Cloudflare Workers. **Default.** |
| [fullstack-elysia-eden](./fullstack-elysia-eden.md) | TanStack Start | Elysia + Eden | — | The alternative, with Eden Treaty |
| [fullstack-convex](./fullstack-convex.md) | TanStack Start | Convex | ✅ | When you want reactivity/realtime out of the box |
| [service-only-hono](./service-only-hono.md) | — | Hono + oRPC | — | A backend service shared by several products (e.g. centralized auth) |
| [mobile-expo](./mobile-expo.md) | Expo RN | (consumes the API) | — | Mobile app on top of the shared domain + API |
| [desktop-electron](./desktop-electron.md) | Electron renderer | IPC / optional API | — | Local-first desktop app |

## The base common to every recipe

Every stack shares the same foundations:

1. **Architecture** — [DDD + hexagonal](../architecture/README.md): domain ← application ← infra, apps only wire things together.
2. **Packages** — the [`infra-*` convention](../packages/infrastructure-naming.md) and the [build strategy](../packages/shared-package-build-strategy.md).
3. **Monorepo** — [Turborepo + Bun workspaces](../monorepos/monorepo-structure.md).
4. **Error handling** — [Result types](../error-handling/result-types.md) and response helpers.
5. **Conventions** — [english-only](../conventions/english-only.md), [schemas-first](../conventions/schemas-first.md), [constants](../conventions/constants-pattern.md), [backlog](../conventions/backlog-pattern.md).

Each recipe assumes this base and only spells out what is specific to the stack.
