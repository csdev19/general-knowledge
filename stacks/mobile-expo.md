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

**3 · API que consume** — según el backend elegido:
- [Cliente isomórfico / API](../api/api-client.md), [gotcha auth cross-origin](../api/gotchas/better-auth-cross-origin.md)

## Notas de ensamblaje (lo específico de este stack)

- **Regla de imports estricta**: la app móvil **solo** importa desde `@scope/domain`.
  Nunca `application` ni `infra-*` (son server-only). El domain es mobile-safe (puro, sin Node).
- **Auth cross-origin**: el cliente móvil no es same-origin; usa el override de origin del
  [gotcha de better-auth](../api/gotchas/better-auth-cross-origin.md). El CORS del server contempla
  `exp://` y `mobile://` justamente para esto.
- **Sin server functions de TanStack Start**: la app habla directo con la API vía el cliente tipado.
- Comparte los mismos [constants](../conventions/constants-pattern.md) y validaciones de dominio que la web.
