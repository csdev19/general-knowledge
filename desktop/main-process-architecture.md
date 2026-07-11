# Main Process Architecture

*Cómo arranca el proceso main de una app de escritorio Electron, qué subsistemas registra, y el ciclo de vida de ventanas + tray.*

Contraparte de [Renderer Architecture](./renderer-architecture.md): cómo está armado el **proceso main** de Electron. El renderer es Chromium puro; el main es el dueño de las ventanas, el tray, los permisos del SO, el acceso a disco y el protocolo de media. Todo lo que toca el sistema operativo vive acá.

## Arranque

`src/main/index.ts` es el entry point. El orden importa:

1. **Antes de `app.whenReady`** — `registerMediaScheme()` registra `app-media://` como esquema privilegiado. Tiene que correr antes del ready o Electron no lo trata como streamable. (`media-protocol.ts`)
2. **En `app.whenReady`** — registra todos los handlers IPC y crea las ventanas:
   - `registerSettings()` — settings persistidos (`settings.json`) + aplica Dock policy y launch-at-login
   - `initMainAnalytics()` + `registerAnalyticsIpc()` — sink de excepciones desde ventanas secundarias
   - handler `app:ready` — flush de acciones diferidas cuando el shell montó sus listeners
   - `registerCaptureSourceHandlers()` — enumeración de pantallas/ventanas
   - `registerMediaHub()` — writer a disco, ventana de la barra, relay de estado (devuelve el handle usado por los guards de cierre/quit)
   - `registerGlobalShortcuts()` — atajos system-wide (start/stop/bring-to-front/capture)
   - `registerLibraryVaultHandlers()` — vault local
   - `registerScreenshotHandlers()` — captura/copy/save + hooks de ventana
   - `registerPermissionHandlers()` — permisos macOS/Windows
   - `registerMediaProtocol()` — streaming de archivos del vault
   - `initAutoUpdater()` — updates en builds empaquetadas
   - `createWindow()` + `new CapturePanelWindow()` + `createTray(...)`
3. **Ciclo de vida** — `before-quit` (con guard de operación activa) destruye el tray y desregistra atajos; `window-all-closed` cierra la app salvo en macOS (donde vive en la menu bar); `activate` (click en el Dock) reabre la ventana principal vía `showMainWindow()`.

## Subsistemas

Cada subsistema es un módulo con una función `register*` que engancha sus handlers. El main no tiene un "god object": `index.ts` solo orquesta el registro.

| Subsistema        | Archivo                            | Qué hace                                                                                                                    |
| ----------------- | ---------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| Window management | `index.ts`                         | `mainWindow` 900×670 (min 720×560), `show` en `ready-to-show`, guards de cierre/quit con operación activa, links externos   |
| Settings store    | `infrastructure/settings-store.ts` | `settings.json` (theme/dock/calidad/atajos/deviceId), Dock policy, launch-at-login, broadcast `settings:changed`            |
| Capture Panel     | `capture-panel-window.ts`          | Ventana flotante frameless del tray (ver abajo)                                                                             |
| Tray              | `tray.ts`                          | Ícono de menu bar: click izq = toggle panel, click der = menú (Open / Quit)                                                 |
| Capture sources   | `capture-sources.ts`               | `desktopCapturer.getSources()` → thumbnails base64 320×200                                                                  |
| Media hub         | `media/media-hub.ts`               | Writer a disco, ventana de la barra, burbuja de cámara, relay estado↔comandos — ver [Media Pipeline](./media-pipeline.md)   |
| Global shortcuts  | `shortcuts/global-shortcuts.ts`    | Atajos system-wide (start/stop/bring-to-front/capture), rebindables desde Settings                                         |
| Screenshots       | `screenshots/`                     | Captura interactiva, copy/save, re-open en el editor                                                                        |
| Permissions       | `permissions.ts`                   | Estado y prompts de screen/mic/camera por plataforma — ver [Permissions & Onboarding](./permissions-and-onboarding.md)     |
| Library vault     | `library/`                         | Archivos en disco + sidecars, cambio de carpeta con validación — ver [Library Vault](./library-vault.md)                    |
| Media protocol    | `media-protocol.ts`                | `app-media://` para reproducir/posters desde el vault                                                                       |
| Auto-updater      | `updater/auto-updater.ts`          | Descarga silenciosa en builds empaquetadas; el renderer muestra el banner de reinicio                                       |
| Analytics         | `services/analytics*.ts`           | Sink de excepciones + forwarding desde ventanas secundarias                                                                 |

El contrato IPC entre main y renderer está en [IPC Contract](./ipc-contract.md).

## Dos ventanas, un renderer

La app no tiene dos bundles de renderer: tiene **uno solo cargado con un query distinto**. `src/renderer/src/main.tsx` lee `?mode=capture` y monta `<CapturePanel/>` en vez de `<App/>` (`main.tsx:10`).

- **Main window** — la app completa (pantalla principal / Library / Settings). Se crea al arrancar y se re-crea on-demand (`showMainWindow` la recrea si fue cerrada).
- **Capture Panel** — `capture-panel-window.ts`: frameless, `alwaysOnTop`, transparente, 340px de ancho, altura 180–640 (inicial 380). Se ancla bajo el ícono del tray (`position()` clampea al work area), se auto-oculta en `blur` (comportamiento natural de popover), y ajusta su alto cuando el renderer reporta el contenido vía `capture-panel:resize`.

## Constraints

- `registerMediaScheme()` **debe** correr antes de `app.whenReady` (esquema privilegiado).
- El tray se mantiene en una variable de módulo y se libera solo en `before-quit` — si no, el GC se lleva el ícono.
- `sandbox: false` en ambas ventanas: el preload necesita `node:*` para el bridge.

## Gotchas

- ⚠️ El **renderer es el mismo** para app y panel; cualquier estado global que asuma una sola instancia se duplica. La página principal y la Capture Panel **no comparten estado en memoria** — coordinan vía hooks/IPC, no vía un store compartido.
- ⚠️ La Capture Panel se oculta en `blur`. Abrir un dropdown/diálogo del SO que robe foco la cierra; tenerlo en cuenta al agregar UI ahí.
- ⚠️ `mainWindow` puede ser `null` (cerrada). Todo acceso pasa por `showMainWindow()`, que la recrea — no asumir que existe.

## Related

- [Renderer Architecture](./renderer-architecture.md) — el otro lado
- [IPC Contract](./ipc-contract.md) — los canales main ↔ renderer
- [Media Pipeline](./media-pipeline.md) — el core de captura/encode
