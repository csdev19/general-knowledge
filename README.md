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
| [effect/](./effect/) | Effect as a default backend tool: why adopt it and when not to, adoption as a decision about **reach** (and the one capability that forces the boundary to move), the lazy-service pattern, and the traps — dishonest error channels, cached failures, runtime ownership, guards that pass for the wrong reason |
| [error-handling/](./error-handling/) | Result types, response helpers, api-response-types, error handlers, retrospectiva de centralización |
| [web/](./web/) | TanStack Start/Router/Query, data loading, server functions, UI package compartido, bundle splitting |
| [api/](./api/) | Hono + oRPC en Cloudflare Workers, patrón api-contract, cliente isomórfico, ADR, gotchas de auth, **servicio de auth centralizado** (topología cross-origin: la tríada CORS/`trustedOrigins`/cookies, KV como caché de sesión) |
| [convex/](./convex/) | Convex como backend reactivo: conexión del cliente (las dos URLs, `useQuery` reactivo, API generada en monorepo) y **Better Auth hosteado dentro de Convex** (versiones SDK 57, fix del `useSession` colgado, `expo-network`, overrides, `exp://`) |
| [mobile/](./mobile/) | Expo / React Native: estructura + domain compartido, **dev builds & Metro** (cuándo recompilar, conexión emulador), **build & install** (debug/release × perfil, `run:*` vs EAS, un install por ambiente, dónde viajan realmente las env vars, **+ kit copy-paste con los scripts completos**) y **Google Maps** (dev build, key, Maps SDK Android + billing, SHA-1) |
| [desktop/](./desktop/) | Electron: main/renderer/preload, IPC tipado, permisos nativos, vault local-first, media pipeline, distribución |
| [infra/](./infra/) | Dominios, DNS y edge routing: **migración de un dominio a Cloudflare Workers** — los registros de parking heredados que rompen el deploy, apex vs `www`, y el checklist de todo lo que invalida un origin nuevo (auth, OAuth, CORS, deep links) |
| [monorepos/](./monorepos/) | Turborepo + Bun workspaces, CI/CD por proyecto, PR checks, release-please, estrategia de testing |
| [packages/](./packages/) | Convención `infra-*`, build strategy (src vs dist), repository contracts, caso de estudio de un package |
| [conventions/](./conventions/) | Constants/enums-as-const, schemas-first, **patrón backlog**, workflow specs+plans, **delegación a agentes** |

## Stacks (recetas listas)

Ver **[stacks/](./stacks/)** para las guías de ensamblaje:

| Stack | Caso de uso |
| --- | --- |
| [fullstack-hono-orpc](./stacks/fullstack-hono-orpc.md) | Web + API type-safe (Hono + oRPC) en Cloudflare Workers. **Default.** |
| [fullstack-elysia-eden](./stacks/fullstack-elysia-eden.md) | Web + API type-safe (Elysia + Eden Treaty) |
| [fullstack-convex](./stacks/fullstack-convex.md) | Fullstack reactivo con Convex (realtime) |
| [service-only-hono](./stacks/service-only-hono.md) | Servicio backend sin cliente, compartido por varios productos (auth centralizado) |
| [mobile-expo](./stacks/mobile-expo.md) | App móvil Expo sobre el domain + API compartidos |
| [desktop-electron](./stacks/desktop-electron.md) | App de escritorio Electron local-first |

## Cómo lo usa un template / proyecto

1. El `README` del proyecto enlaza al stack elegido: `stacks/<stack>.md`.
2. La receta lo lleva, en orden, por los docs de conocimiento plano que necesita.
3. Nada de copiar y pegar docs entre proyectos — se enlaza a este hub.

## Convenciones que vale la pena adoptar desde el día 1

- **[Patrón backlog](./conventions/backlog-pattern.md)** — carpeta `backlog/` con el mapa
  de "lo que viene"; nunca pierdes trabajo diferido.
- **[Patrón changelog](./conventions/changelog-pattern.md)** — diario de decisiones en el
  docs app: un archivo por entrada, índice auto-generado (cero merge conflicts),
  complementa el `CHANGELOG.md` de release-please.
- **[Workflow specs + plans](./conventions/specs-and-plans-workflow.md)** — brainstorm →
  spec (diseño) → plan (implementación) → archivo al shippear.
- **[Plan → backlog](./conventions/plan-to-backlog.md)** — convertir un plan aprobado en
  entregables autosuficientes del backlog, ejecutables por agentes en paralelo.
- **[Delegación a agentes](./conventions/ai-agent-delegation.md)** — los dos modos
  (rápido/caro vs lento/barato), qué roles nunca se abaratan, y la evidencia medida de
  que lo que hace lento un trabajo delegado es la estructura del proceso, no el tier
  del modelo.

---

_El diseño de este hub está en [`docs/specs/2026-07-10-general-knowledge-hub-design.md`](./docs/specs/2026-07-10-general-knowledge-hub-design.md)._
