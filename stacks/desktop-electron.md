# Stack: Desktop Electron

_App de escritorio Electron, local-first, con arquitectura main/renderer/preload y un
IPC tipado. Opcionalmente sincroniza con una API remota._

## Cuándo usarlo

Cuando necesitas acceso a APIs nativas del sistema (captura de pantalla/audio, filesystem,
permisos del OS, bandeja, auto-update) y una experiencia local-first. Ver el ADR de framework
antes de decidir entre Electron y Tauri.

## Lista de lectura (en orden)

**1 · Decisión de framework**
- [Electron vs Tauri (ADR)](../desktop/electron-vs-tauri.md) — cuándo cada uno, dónde está el gap real

**2 · Arquitectura desktop**
- [Main process](../desktop/main-process-architecture.md) — boot, subsistemas, ciclo de vida
- [Renderer](../desktop/renderer-architecture.md) — estructura de UI, migración de pantallas
- [IPC contract](../desktop/ipc-contract.md) — canales tipados main↔renderer
- [Permisos y onboarding](../desktop/permissions-and-onboarding.md) — permisos nativos cross-platform
- [Library / vault local-first](../desktop/library-vault.md) — storage en filesystem + metadata sidecars
- [Media / compute pipeline](../desktop/media-pipeline.md) — pipeline pesado fuera de React
- [Filesystem-first & monetización](../desktop/filesystem-first-monetization.md)
- [Analytics y feature flags](../desktop/analytics-and-flags.md), [filosofía de producto](../desktop/product-philosophy.md)

**3 · Cimientos compartidos**
- [Arquitectura DDD + hexagonal](../architecture/README.md) — el domain sigue siendo puro y reutilizable
- [Packages `infra-*`](../packages/infrastructure-naming.md), [monorepo](../monorepos/monorepo-structure.md)
- Si sincroniza con backend: [API](../api/README.md) y [error handling](../error-handling/README.md)

## Notas de ensamblaje (lo específico de este stack)

- **Tres procesos, un contrato**: main (Node), renderer (React) y preload (bridge). Todo el
  IPC pasa por canales **tipados** — ver [ipc-contract](../desktop/ipc-contract.md).
- **Lo pesado vive fuera de React**: el pipeline de captura/encode/compute corre en el main
  o en workers, escribe a disco de forma posicional y publica al renderer vía un hub.
- **Local-first**: el vault en filesystem + metadata sidecars es la fuente de verdad; la sync
  remota (si existe) es secundaria. Habilita monetización sin backend obligatorio.
- **Node externals**: el main es Node — cuida el bundling (externalizar deps nativas). Ver la
  lección en [packages/case-study](../packages/case-study-design-tokens-package.md) y el gotcha de auth.
- **Distribución**: auto-update (electron-updater) + version gate; documentado en el backlog/roadmap del producto.
