# General Knowledge

Hub central de conocimiento reutilizable para construir productos: arquitectura,
stacks (web / api / mobile / desktop), manejo de errores, monorepos y packages.

La idea: **una sola fuente de verdad**. Los templates y proyectos nuevos **enlazan
aquí** en lugar de arrastrar copias de la documentación. Cuando eliges un stack,
ya está todo preparado — solo lo consumes.

## Cómo está organizado

Dos capas:

- **Conocimiento plano por tema** — la fuente de verdad, reutilizable y (en su mayoría)
  agnóstica del stack. Cada carpeta tiene su propio `README.md` índice.
- **[`stacks/`](./stacks/)** — recetas que **componen** los temas planos en una guía
  "cómo construir con este stack". No duplican contenido: enlazan + agregan las notas
  de ensamblaje propias de cada stack. **A esto apunta el README de un template.**

## Temas

| Tema | Qué contiene |
| --- | --- |
| [architecture/](./architecture/) | DDD + hexagonal, bounded contexts, shared kernel, repository pattern, domain modeling, observabilidad, security hardening, ADRs |
| [error-handling/](./error-handling/) | Result types, response helpers, api-response-types, error handlers, retrospectiva de centralización |
| [web/](./web/) | TanStack Start/Router/Query, data loading, server functions, UI package compartido, bundle splitting |
| [api/](./api/) | Hono + oRPC en Cloudflare Workers, patrón api-contract, cliente isomórfico, ADR, gotchas de auth |
| [convex/](./convex/) | Convex como backend reactivo: conexión del cliente (las dos URLs, `useQuery` reactivo, API generada en monorepo) y **Better Auth hosteado dentro de Convex** (versiones SDK 57, fix del `useSession` colgado, `expo-network`, overrides, `exp://`) |
| [mobile/](./mobile/) | Expo / React Native: estructura + domain compartido, **dev builds & Metro** (cuándo recompilar, conexión emulador), y **Google Maps** (dev build, key, Maps SDK Android + billing, SHA-1) |
| [desktop/](./desktop/) | Electron: main/renderer/preload, IPC tipado, permisos nativos, vault local-first, media pipeline, distribución |
| [monorepos/](./monorepos/) | Turborepo + Bun workspaces, CI/CD por proyecto, PR checks, release-please, estrategia de testing |
| [packages/](./packages/) | Convención `infra-*`, build strategy (src vs dist), repository contracts, caso de estudio de un package |
| [conventions/](./conventions/) | Constants/enums-as-const, schemas-first, **patrón backlog**, workflow specs+plans |

## Stacks (recetas listas)

Ver **[stacks/](./stacks/)** para las guías de ensamblaje:

| Stack | Caso de uso |
| --- | --- |
| [fullstack-hono-orpc](./stacks/fullstack-hono-orpc.md) | Web + API type-safe (Hono + oRPC) en Cloudflare Workers. **Default.** |
| [fullstack-elysia-eden](./stacks/fullstack-elysia-eden.md) | Web + API type-safe (Elysia + Eden Treaty) |
| [fullstack-convex](./stacks/fullstack-convex.md) | Fullstack reactivo con Convex (realtime) |
| [mobile-expo](./stacks/mobile-expo.md) | App móvil Expo sobre el domain + API compartidos |
| [desktop-electron](./stacks/desktop-electron.md) | App de escritorio Electron local-first |

## Cómo lo usa un template / proyecto

1. El `README` del proyecto enlaza al stack elegido: `stacks/<stack>.md`.
2. La receta lo lleva, en orden, por los docs de conocimiento plano que necesita.
3. Nada de copiar y pegar docs entre proyectos — se enlaza a este hub.

## Convenciones que vale la pena adoptar desde el día 1

- **[Patrón backlog](./conventions/backlog-pattern.md)** — carpeta `backlog/` con el mapa
  de "lo que viene"; nunca pierdes trabajo diferido.
- **[Workflow specs + plans](./conventions/specs-and-plans-workflow.md)** — brainstorm →
  spec (diseño) → plan (implementación) → archivo al shippear.

---

_El diseño de este hub está en [`docs/specs/2026-07-10-general-knowledge-hub-design.md`](./docs/specs/2026-07-10-general-knowledge-hub-design.md)._
