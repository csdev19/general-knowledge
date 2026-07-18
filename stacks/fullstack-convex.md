# Stack: Fullstack Convex

_Web (TanStack Start) sobre Convex como backend reactivo. Realtime, queries reactivas y
persistencia sin montar tu propia capa API/DB._

## Cuándo usarlo

Cuando el realtime/reactividad es un requisito de primera clase (colaboración en vivo,
listas que se actualizan solas, presencia) y quieres evitar el overhead de API + DB +
websockets manuales. A cambio, cedes parte del control de la capa de infraestructura.

Si no necesitas realtime, el default sigue siendo [hono-orpc](./fullstack-hono-orpc.md).

## Lista de lectura (en orden)

**1 · Cimientos que siguen aplicando**
- [Arquitectura DDD + hexagonal](../architecture/README.md) — el domain sigue siendo puro y compartible
- [Domain modeling strategy](../architecture/domain-modeling-strategy.md)
- [schemas-first](../conventions/schemas-first.md) — los schemas Zod siguen siendo la fuente de verdad
- [Monorepo](../monorepos/monorepo-structure.md)

**2 · Convex** — [conexión del cliente](../convex/client-connection.md) (las dos URLs, `useQuery` reactivo, API generada), [Better Auth en Convex](../convex/better-auth.md)

**3 · Web** — [data loading](../web/data-loading.md), [UI package](../web/web-ui-package.md), [bundle splitting](../web/bundle-splitting.md), [local preview pre-prod](../web/local-preview.md)

**4 · Mobile (opcional)** — si sumas app Expo sobre este Convex: [mobile-expo](./mobile-expo.md), [dev builds & Metro](../mobile/expo-dev-builds-and-metro.md)

**5 · Convenciones** — [constants](../conventions/constants-pattern.md), [backlog](../conventions/backlog-pattern.md)

## Notas de ensamblaje (lo específico de este stack)

- **Convex reemplaza `infra-db` + la capa API**: las Convex functions (queries/mutations/actions)
  cumplen el rol de repositorios + use cases expuestos. El **domain sigue viviendo en su package**
  y se importa desde las funciones Convex.
- **Reactividad**: las queries de Convex son suscripciones; el cliente se re-renderiza solo.
  No necesitas TanStack Query para el estado servidor reactivo (sí para lo que quede fuera de Convex).
- **Auth-in-Convex**: Better Auth corre **dentro** de la deployment (`@convex-dev/better-auth`),
  sirviendo `/api/auth/*` en su HTTP router — no hay server API aparte ni proxy same-origin de
  Workers. El setup completo, las **versiones pinneadas** (crítico en Expo SDK 57) y los gotchas
  están en [convex/better-auth](../convex/better-auth.md). Alternativa: auth en un server externo
  que Convex valida por `customJwt` (ver ese doc).
- **`router.invalidate()`** tras cambios de sesión sigue siendo necesario para el `beforeLoad` del root.
- **Preview pre-prod (Convex = variante sin API worker)**: el build de prod difiere del `dev`
  (split de CSS + orden de carga con logical properties de Tailwind v4, `web-ui` resolviendo a
  `dist` en vez de `src`, minificación). Como Convex corre su propio `convex dev`, el `preview:local`
  **no arranca ningún worker de API** — solo `build web-ui → build web → vite preview`. Detalle y el
  porqué en [local preview pre-prod](../web/local-preview.md) (Variante B).
- Para patrones de bajo nivel de Convex (functions, realtime, migraciones, cron, file storage),
  usa las skills `convex-*` del entorno; el hub cubre conexión, auth y arquitectura/dominio.

> Conocimiento plano de Convex: [`convex/`](../convex/) — conexión del cliente y Better Auth
> hosteado en Convex.
