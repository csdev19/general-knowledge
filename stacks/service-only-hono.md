# Stack: Service-only Hono (backend sin cliente)

_Un servicio backend desplegado como un único Cloudflare Worker (Hono + oRPC), sin app web
ni móvil en el monorepo. Los consumidores viven en **otros repos** y lo llaman por HTTPS.
El caso canónico es un servicio de auth centralizado._

## Cuándo usarlo

Cuando una capacidad tiene que ser compartida por varios productos y no puede vivir dentro
de ninguno de ellos: identidad, facturación, notificaciones. Si el servicio solo lo consume
un producto, no lo separes — usa [fullstack-hono-orpc](./fullstack-hono-orpc.md), que además
te da cookies same-origin gratis.

La diferencia estructural con los demás stacks es que **no hay proxy ni Service Binding**:
todo consumidor llega cross-origin. Eso cambia auth, cookies y CORS de "detalle" a
"restricción de diseño" — ver [servicio de auth centralizado](../api/centralized-auth-service.md).

## Lista de lectura (en orden)

**1 · Cimientos**
- [Arquitectura DDD + hexagonal](../architecture/README.md) — la regla de dependencias
- [Domain modeling strategy](../architecture/domain-modeling-strategy.md) — cuánta ceremonia aplicar
- [Repository pattern](../architecture/repository-pattern.md) — interfaz en domain, impl en infra
- [Estructura de monorepo](../monorepos/monorepo-structure.md) — Turborepo + Bun workspaces
- [Convención `infra-*`](../packages/infrastructure-naming.md) y [build strategy](../packages/shared-package-build-strategy.md)

**2 · API (Hono + oRPC)**
- [Hono + oRPC overview](../api/hono.md) — el stack de servidor (ignora la parte de web/proxy)
- [Patrón api-contract](../api/api-contract-pattern.md) — dónde vive el contrato
- [ADR: oRPC + Hono + Cloudflare](../api/decisions/adr-0002-orpc-hono-cloudflare.md) — por qué esta capa

**3 · El servicio cross-origin (lo propio de este stack)**
- [Servicio de auth centralizado](../api/centralized-auth-service.md) — topología, la tríada CORS/`trustedOrigins`/cookies, KV como caché de sesión
- [Gotcha: better-auth cross-origin](../api/gotchas/better-auth-cross-origin.md) — headers inmutables en Workers
- [Wrangler y config de env](../monorepos/wrangler-env-config.md) — `.env` como única fuente de verdad
- [Migración de dominio a Workers](../infra/custom-domain-migration.md) — pon servicio y consumidores bajo un mismo dominio padre

**4 · Errores y observabilidad**
- [Result types](../error-handling/result-types.md) → [response helpers](../error-handling/response-helpers.md) → [api-response-types](../error-handling/api-response-types.md) → [error handlers](../error-handling/error-handlers.md)
- [Observabilidad](../architecture/observability.md) y [security hardening](../architecture/security-hardening.md)

**5 · CI/CD y testing**
- [CI/CD por proyecto](../monorepos/ci-cd-pipelines.md), [PR checks](../monorepos/pr-checks.md), [release-please](../monorepos/release-automation.md)
- [Estrategia de testing](../monorepos/testing-strategy.md)

## Notas de ensamblaje (lo específico de este stack)

- **Un solo Worker.** El app *es* el backend, así que el workflow de deploy tiene un único
  `wrangler deploy`; no hay paso de Pages ni de build de cliente.
- **CORS es el límite de acceso real.** No hay proxy que lo esconda. La allowlist sale de
  `CORS_ORIGIN` y alimenta también `trustedOrigins`; con `credentials: true` el wildcard es
  ilegal. Dar de alta un consumidor = una entrada más en esa variable.
- **Cookies `sameSite: "none"` + `secure`.** Y pon servicio y consumidores bajo un mismo
  dominio padre (`auth.example.com` / `app.example.com`) para que sigan siendo same-site
  pese a las políticas de cookies de terceros.
- **Sin React en ningún package.** `i18n` sirve copy transaccional (correos, push, mensajes
  de error) vía el core no-React de `use-intl`; fuera provider, hooks y peer dep de React.
- **`application` puede nacer vacío.** Un barrel con `export {}` que marca dónde van los
  casos de uso es scaffolding legítimo, no residuo — documenta que es intencional.
- **Prefija las tablas** y acota `tablesFilter` a ese prefijo, para que `drizzle-kit push`
  nunca proponga borrar tablas de otro servicio que comparta la base.
- **Bindings en `wrangler.jsonc`, config en `.env`.** KV, Durable Objects y colas son
  bindings; secretos y URLs no. Nunca un bloque `vars`.

## Generarlo

`bun run customize` en [monorepo-template](https://github.com/niway-dev/monorepo-template)
ofrece este stack como **Backend only — Hono + oRPC API**: conserva `apps/server-hono` y
`packages/i18n`, borra web, móvil, `web-ui` y `tokens`, y genera el workflow de deploy de
un solo Worker.
