# Stack: Fullstack Hono + oRPC

_Web (TanStack Start) + API (Hono + oRPC) desplegados como dos Cloudflare Workers,
conectados por Service Bindings. Type-safety de extremo a extremo. **Stack default.**_

## Cuándo usarlo

Tu default para productos web fullstack: API tipada, cookies de auth same-origin sobre
Cloudflare, contrato compartido sin publicar un package. Elige este salvo que necesites
realtime out-of-the-box (→ [convex](./fullstack-convex.md)) o prefieras Eden Treaty
(→ [elysia](./fullstack-elysia-eden.md)).

## Lista de lectura (en orden)

**1 · Cimientos**
- [Arquitectura DDD + hexagonal](../architecture/README.md) — la regla de dependencias
- [Domain modeling strategy](../architecture/domain-modeling-strategy.md) — cuánta ceremonia aplicar
- [Repository pattern](../architecture/repository-pattern.md) — interfaz en domain, impl en infra
- [Estructura de monorepo](../monorepos/monorepo-structure.md) — Turborepo + Bun workspaces
- [Convención `infra-*`](../packages/infrastructure-naming.md) y [build strategy](../packages/shared-package-build-strategy.md)

**2 · API (Hono + oRPC)**
- [Hono + oRPC overview](../api/hono.md) — el stack de servidor completo
- [Patrón api-contract](../api/api-contract-pattern.md) — dónde vive el contrato y cómo se consume
- [Cliente isomórfico](../api/api-client.md) — mismo cliente en server y browser
- [ADR: oRPC + Hono + Cloudflare](../api/decisions/adr-0002-orpc-hono-cloudflare.md) — por qué esta capa
- [Gotcha: better-auth cross-origin](../api/gotchas/better-auth-cross-origin.md) — headers inmutables en Workers

**3 · Web (TanStack Start)**
- [Data loading](../web/data-loading.md) — loaders + Query; server functions vs cliente isomórfico
- [Server functions](../web/server-functions.md) — el boundary y sus reglas
- [UI package compartido](../web/web-ui-package.md) — shadcn/ui + theming OKLCH
- [Bundle splitting](../web/bundle-splitting.md) y [preview local](../web/local-preview.md)

**4 · Errores y observabilidad**
- [Result types](../error-handling/result-types.md) → [response helpers](../error-handling/response-helpers.md) → [api-response-types](../error-handling/api-response-types.md) → [error handlers](../error-handling/error-handlers.md)
- [Observabilidad](../architecture/observability.md) y [security hardening](../architecture/security-hardening.md)

**5 · CI/CD y testing**
- [CI/CD por proyecto](../monorepos/ci-cd-pipelines.md), [PR checks](../monorepos/pr-checks.md), [release-please](../monorepos/release-automation.md)
- [Estrategia de testing](../monorepos/testing-strategy.md)

## Notas de ensamblaje (lo específico de este stack)

- **Proxy same-origin**: el Worker web proxya `/api/auth/*` y `/api/v1/*` al Worker API.
  En producción vía Service Bindings (Worker-a-Worker, sin DNS público); en dev local, `fetch()`.
  Los `Set-Cookie` se reescriben para quitar `domain=` y asignar la cookie al dominio web.
- **El contrato vive en la app de servidor** (`apps/api/src/contract/`), no en un package aparte.
  La web lo importa por ruta relativa cross-app (mismo patrón que `type { App }` de Eden).
  Ver [api-contract-pattern](../api/api-contract-pattern.md) si algún día quieres extraerlo a package.
- **`router.invalidate()`** tras sign-in/up/out para refrescar el `beforeLoad` del root; si no,
  el header muestra estado de auth viejo.
- **La web NO corre Better Auth localmente** — proxya al auth del backend. `infra-auth` no es
  dependencia de la app web.
- **CORS del server solo para mobile** (`exp://`, `mobile://`); la web es same-origin vía proxy.
