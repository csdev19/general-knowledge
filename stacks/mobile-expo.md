# Stack: Mobile Expo

_App móvil Expo / React Native que consume el mismo domain y la misma API que la web._

## Cuándo usarlo

Cuando necesitas una app móvil nativa que comparta modelo de dominio, schemas y contrato
con el resto del monorepo. Se suma a un stack web/API existente, no lo reemplaza.

## Lista de lectura (en orden)

**1 · Base compartida**
- [Arquitectura DDD + hexagonal](../architecture/README.md) — la app móvil **solo** importa `@scope/domain`
- [schemas-first](../conventions/schemas-first.md) — mismos schemas Zod que web/API
- [Estructura de monorepo](../monorepos/monorepo-structure.md)

**2 · Mobile**
- [Mobile app](../mobile/mobile-app.md) — estructura Expo, consumo del domain, auth y API
- [Dev builds & Metro](../mobile/expo-dev-builds-and-metro.md) — **cómo correrla**: native shell vs JS, Expo Go vs dev build, cuándo recompilar, conexión Metro en emulador, setup de entorno
- [Google Maps](../mobile/google-maps.md) — si usas mapas (dev build, key, Maps SDK Android + billing, SHA-1)

**3 · Backend que consume** — según el elegido:
- **Convex** (data + auth en un backend): [conexión del cliente](../convex/client-connection.md), [Better Auth en Convex](../convex/better-auth.md)
- **API externa** (Hono/Elysia): [cliente isomórfico](../api/api-client.md), [gotcha auth cross-origin](../api/gotchas/better-auth-cross-origin.md)

## Notas de ensamblaje (lo específico de este stack)

- **Regla de imports estricta**: la app móvil **solo** importa desde `@scope/domain` (y el
  `_generated/api` del backend Convex si aplica). Nunca `application` ni `infra-*` (server-only).
- **Correr ≠ Expo Go**: en cuanto uses un módulo nativo que Expo Go no trae (mapas, config
  propia) necesitas un **dev build**. La [guía de dev builds & Metro](../mobile/expo-dev-builds-and-metro.md)
  es la referencia — incluye la regla "rebuild vs `bun dev`" y la conexión Metro del emulador.
- **Auth por origin del cliente móvil**:
  - Con **Convex**: `trustedOrigins` incluye `native://` + **`exp://`** (Expo Go); ver [Better Auth en Convex](../convex/better-auth.md).
  - Con **API externa**: el override de origin del [gotcha de better-auth](../api/gotchas/better-auth-cross-origin.md); el CORS contempla `exp://` / `mobile://`.
- **Versiones (Expo SDK 57)**: el stack Convex+Better Auth es frágil por versión — pinnea las
  del [doc de auth](../convex/better-auth.md) o el login se cuelga en un black screen.
- Comparte los mismos [constants](../conventions/constants-pattern.md) y validaciones de dominio que la web.
