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

**2 · Web** — [data loading](../web/data-loading.md), [UI package](../web/web-ui-package.md)

**3 · Convenciones** — [constants](../conventions/constants-pattern.md), [backlog](../conventions/backlog-pattern.md)

## Notas de ensamblaje (lo específico de este stack)

- **Convex reemplaza `infra-db` + la capa API**: las Convex functions (queries/mutations/actions)
  cumplen el rol de repositorios + use cases expuestos. El **domain sigue viviendo en su package**
  y se importa desde las funciones Convex.
- **Reactividad**: las queries de Convex son suscripciones; el cliente se re-renderiza solo.
  No necesitas TanStack Query para el estado servidor reactivo (sí para lo que quede fuera de Convex).
- **Auth**: Better Auth se integra vía el adaptador de Convex; ya no aplica el proxy same-origin
  de Workers del stack default.
- **`router.invalidate()`** tras cambios de sesión sigue siendo necesario para el `beforeLoad` del root.
- Para patrones detallados de Convex (functions, realtime, migraciones, cron, file storage),
  usa las skills `convex-*` del entorno de desarrollo — este hub cubre la parte de arquitectura/dominio.

> Nota: este hub aún no tiene una carpeta `convex/` de conocimiento plano. Si acumulas
> suficientes patrones propios, créala y enlázala aquí (Fase futura).
