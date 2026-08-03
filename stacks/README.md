# Stacks

Recetas de ensamblaje. Cada archivo describe **cómo construir con un stack concreto**
y **enlaza** a los docs de conocimiento plano (`../architecture/`, `../web/`, `../api/`, …)
que lo componen. No duplican contenido: son una lista de lectura ordenada + las notas
de ensamblaje propias del stack.

**El README de un template/proyecto apunta a la receta de su stack.**

| Stack | Web | API | Realtime | Caso de uso |
| --- | --- | --- | --- | --- |
| [fullstack-hono-orpc](./fullstack-hono-orpc.md) | TanStack Start | Hono + oRPC | — | Fullstack type-safe en Cloudflare Workers. **Default.** |
| [fullstack-elysia-eden](./fullstack-elysia-eden.md) | TanStack Start | Elysia + Eden | — | Alternativa con Eden Treaty |
| [fullstack-convex](./fullstack-convex.md) | TanStack Start | Convex | ✅ | Cuando quieres reactividad/realtime out-of-the-box |
| [service-only-hono](./service-only-hono.md) | — | Hono + oRPC | — | Servicio backend compartido por varios productos (p. ej. auth centralizado) |
| [mobile-expo](./mobile-expo.md) | Expo RN | (consume API) | — | App móvil sobre el domain + API compartidos |
| [desktop-electron](./desktop-electron.md) | Electron renderer | IPC / opcional API | — | App de escritorio local-first |

## Base común a todas las recetas

Todos los stacks comparten los mismos cimientos:

1. **Arquitectura** — [DDD + hexagonal](../architecture/README.md): domain ← application ← infra, apps solo cablean.
2. **Packages** — [convención `infra-*`](../packages/infrastructure-naming.md) y [build strategy](../packages/shared-package-build-strategy.md).
3. **Monorepo** — [Turborepo + Bun workspaces](../monorepos/monorepo-structure.md).
4. **Error handling** — [Result types](../error-handling/result-types.md) y helpers de respuesta.
5. **Convenciones** — [schemas-first](../conventions/schemas-first.md), [constants](../conventions/constants-pattern.md), [backlog](../conventions/backlog-pattern.md).

Cada receta asume esta base y solo detalla lo específico del stack.
