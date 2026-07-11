# Stack: Fullstack Elysia + Eden Treaty

_Web (TanStack Start) + API (Elysia) con cliente type-safe vía Eden Treaty. Misma base
DDD/hexagonal y mismo modelo de proxy same-origin que el stack default._

## Cuándo usarlo

Cuando prefieres Elysia + Eden Treaty (type-safety derivada del `type { App }` exportado)
en lugar de oRPC contract-first. Ergonomía distinta, mismos cimientos. Si dudas, el
default es [hono-orpc](./fullstack-hono-orpc.md).

## Lista de lectura (en orden)

**1 · Cimientos** — idénticos al default:
- [Arquitectura DDD + hexagonal](../architecture/README.md), [domain modeling](../architecture/domain-modeling-strategy.md), [repository pattern](../architecture/repository-pattern.md)
- [Monorepo](../monorepos/monorepo-structure.md), [packages `infra-*`](../packages/infrastructure-naming.md)

**2 · API (Elysia + Eden)**
- [Cliente isomórfico](../api/api-client.md) — aplica igual; con Eden el cliente se deriva del `App` type
- [Gotcha: better-auth cross-origin](../api/gotchas/better-auth-cross-origin.md)

**3 · Web** — igual que el default:
- [Data loading](../web/data-loading.md), [server functions](../web/server-functions.md), [UI package](../web/web-ui-package.md)

**4 · Errores** — [Result types](../error-handling/result-types.md) + [helpers](../error-handling/response-helpers.md) + [handlers](../error-handling/error-handlers.md)

**5 · CI/CD y testing** — [pipelines](../monorepos/ci-cd-pipelines.md), [testing](../monorepos/testing-strategy.md)

## Notas de ensamblaje (lo específico de este stack)

- **Type-safety por `App` type**: el server exporta `export type App = typeof app` y la web
  construye el cliente Eden con `treaty<App>()`. No hay contrato separado; el tipo ES el contrato.
- **El cliente Eden usa `window.location.origin`** (no la URL del server) para ir por el proxy same-origin.
- **Mismo proxy que el default**: Worker web → Worker API vía Service Bindings, reescritura de `Set-Cookie`.
- Todo lo demás (auth por proxy, `router.invalidate()`, CORS solo mobile) es igual al
  [stack default](./fullstack-hono-orpc.md) — revisa sus notas de ensamblaje.
