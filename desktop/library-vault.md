# Library Vault

*El modelo de almacenamiento local-first — layout en disco, sidecars de metadata, ubicación configurable del vault, y el protocolo de media privilegiado para reproducir.*

El modelo de datos local-first de la app. Los artefactos generados son **archivos en una carpeta del usuario** — no hay base de datos. El archivo es la fuente de verdad; la metadata vive en sidecars al lado. Esto es la implementación técnica de la filosofía en [Filesystem-First Monetization](./filesystem-first-monetization.md).

## Layout en disco

```
<vault>/                          (default: ~/Videos/<App>)
├── 2026-06-14 Sprint Review.mp4   ← el archivo ES la fuente de verdad
├── 2026-06-13 Bug repro.mp4
└── .app/                          ← metadata de la app (oculta)
    ├── <id>.json                   ← sidecar: title, durationSeconds, createdAt
    └── <id>.jpg                    ← poster/thumbnail (best-effort)
```

**El filename SIN extensión es el `id`.** `2026-06-14 Sprint Review.mp4` → id = `2026-06-14 Sprint Review`. No hay tabla de IDs; el filesystem es el índice.

`LibraryVault` (`library/library-vault.ts:30`) es una clase **pura**: recibe el directorio inyectado, sin tocar `electron`. Por eso es testeable contra una carpeta temporal.

## Formato de archivo

`VIDEO_EXTS = [".mp4", ".webm"]`, con `.mp4` primero (`library-vault.ts:7`). Al resolver un id se prueba `.mp4` y luego `.webm`; si ninguno existe todavía, se asume `.mp4`. El `.mp4` es lo que el pipeline actual escribe. El `.webm` queda soportado para artefactos viejos.

## Sidecar de metadata

`.app/<id>.json`, todos los campos opcionales:

```json
{ "title": "Sprint Review", "durationSeconds": 312, "createdAt": 1749938548133 }
```

`describe()` mergea el sidecar con `stat()` del archivo y rellena defaults: si falta `title`, humaniza el id; si falta `createdAt`, usa `birthtimeMs`; `durationSeconds` por defecto 0 (`library-vault.ts:71`). `writeMeta()` mergea (no sobrescribe) — `rename` solo toca `title`, deja el resto intacto.

## Operaciones

| Op                  | Qué hace                                                                        | Toca el archivo                    |
| ------------------- | ------------------------------------------------------------------------------- | ---------------------------------- |
| `list()`            | readdir → filtra exts → `describe` cada uno → ordena por `createdAt` desc        | no                                 |
| `describe(id)`      | `stat` + sidecar → `LocalRecording` (o `null` si no hay archivo)                | no                                 |
| `rename(id, title)` | escribe `title` en el sidecar                                                   | **no** — el archivo no se renombra |
| `remove(id)`        | borra archivo + sidecar + thumbnail (best-effort, `allSettled`)                 | sí                                 |

`thumbnailUrl` apunta a `app-media://thumb/<id>` solo si el `.jpg` existe.

## Ubicación del vault

`library/vault-location.ts` — store de preferencias minúsculo en `preferences.json` dentro de `userData` (sin dependencia `electron-store`):

- **Default**: `app.getPath("videos")/<App>`, con fallback a `~/Videos/<App>`.
- **Custom**: si el usuario eligió carpeta, se guarda en `preferences.json` y `VaultDirectory.isCustom = true`.
- `chooseVaultDirectory` abre un picker nativo; `resetVaultDirectory` borra la preferencia y vuelve al default.

Cada handler crea un `LibraryVault` fresco ligado al directorio **actual** (`currentVault()` en `library/index.ts:9`), así un cambio de carpeta aplica inmediato sin reiniciar.

## Reproducción: esquema de media privilegiado

El renderer no puede leer `file://` directo (CSP + seguridad). El main expone un **esquema privilegiado streamable** (`media-protocol.ts`), aquí `app-media://`:

- `app-media://recording/<id>` → el archivo, Range-aware (necesario para seek en el `<video>`).
- `app-media://thumb/<id>` → el poster `.jpg`.

`registerMediaScheme()` corre antes de `app.whenReady`; `registerMediaProtocol()` después. El handler mapea el id a su path real vía `recordingFilePath`/`thumbnailFilePath` y hace `net.fetch(file://…)`.

## Gotchas

- ⚠️ **El id es el filename.** Renombrar el `title` NO renombra el archivo (intencional: el path es estable, la metadata es mutable). Si el usuario renombra el `.mp4` en el explorador de archivos, cambia el id y se "desconecta" del sidecar viejo.
- ⚠️ `isUnsafeId()` (`media-protocol.ts:13`) rechaza ids con `/`, `\` o `..` — guard contra path traversal en el protocolo. No bypassear.
- ⚠️ El vault directory puede cambiar en runtime; nunca cachear un `LibraryVault` de larga vida — usar `currentVault()`.
- ⚠️ Borrar es best-effort (`Promise.allSettled`): si el sidecar o thumb fallan, el archivo igual se borra. No tratar un fallo parcial como "no se borró nada".

## Related

- [Filesystem-First Monetization](./filesystem-first-monetization.md) — el porqué de producto
- [IPC Contract](./ipc-contract.md) — los handlers `library:*`
