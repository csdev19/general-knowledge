# Filesystem-First Architecture + Monetización

*ADR — por qué una app local-first pone el filesystem como fuente de verdad, la estructura del vault con sidecars atómicos, y por qué local-first no dificulta monetizar.*

## Veredicto (TL;DR)

**Arquitectura:** Filesystem-first. El bug que perdió 200 GB de archivos intactos por un JSON corrupto es la prueba de que el store no puede ser el portero.

**Monetización:** Filesystem-first **no** dificulta monetizar — es un diferenciador. Hay precedentes de apps open-source, local-first, "trae tu propio bucket S3", con decenas de miles de estrellas en GitHub, que cobran una suscripción mensual. Lo que se vende es la **capa de nube/sharing**, nunca las features locales.

```
Lo local es el producto y el diferenciador (gratis).
Lo que se cobra es la capa de red: sharing, streaming, colaboración, hosting.
Nunca cobrar por features locales en un modelo local-first.
```

---

## La arquitectura unificada

### El principio

```
Un artefacto no lo CREA la app.
Un artefacto lo DESCUBRE la app.

Filesystem = fuente de verdad
Metadata    = cache (regenerable)
```

Reglas sin excepciones:

1. **Si el archivo existe, el artefacto existe.** Siempre.
2. **Si la metadata desaparece, el artefacto sigue existiendo.**
3. **La metadata se regenera; los archivos no.**

### Estructura del vault

```
~/Movies/<App>/                   ← el vault
├── 2026-06-14 Sprint Review.webm   ← la fuente de verdad
├── 2026-06-13 Bug repro.webm
└── .app/                            ← todo lo de la app, oculto
    ├── meta/
    │   ├── rec_a8Kf2mNp.json         ← sidecar por artefacto
    │   └── rec_Qx91dL23.json
    ├── thumbnails/
    └── index.json                    ← cache DERIVADO, siempre reconstruible
```

**Por qué la metadata va DENTRO del vault:** Si va en `electron-store` (fuera de la carpeta), el usuario sincroniza archivos por Dropbox/iCloud pero los títulos y tags **no viajan**. Obsidian no comete ese error: pone su metadata en `.obsidian/` _dentro_ del vault. Aplicá el mismo patrón con una carpeta oculta `.app/`.

### Sidecars, no un índice único

Un `.json` por artefacto en `.app/meta/`. Corromper un archivo te cuesta los tags de _uno_, no de toda la librería.

### Escritura atómica

```ts
async function writeSidecarAtomic(targetPath: string, data: object) {
  const tmp = path.join(path.dirname(targetPath), `.${path.basename(targetPath)}.tmp`)
  await writeFile(tmp, JSON.stringify(data, null, 2))
  await rename(tmp, targetPath) // rename es atómico en el mismo filesystem
}
```

Un crash a mitad de escritura nunca deja un JSON corrupto.

### Identidad: UID por birthtimeMs

```
fileUID = local-{birthtimeMs}   // e.g. local-1749938548133
```

`stat.birthtime` se preserva en macOS al renombrar. El UID es estable aunque el usuario renombre el archivo en el explorador — la metadata sobrevive.

### Sidecar de ejemplo

```json
{
  "id": "rec_a8Kf2mNp",
  "title": "Sprint Review",
  "createdAt": "2026-06-14T15:32:00Z",
  "fileName": "2026-06-14 Sprint Review.webm",
  "fingerprint": {
    "size": 184320000,
    "durationMs": 312000,
    "headHash": "sha256:9f2c…"
  },
  "share": {
    "state": "private",
    "url": null,
    "uploadedAt": null
  },
  "schemaVersion": 1
}
```

### Startup recovery

```
main.ts startup:
  1. migrateElectronStoreIfNeeded()   — one-time: old JSON → sidecars
  2. recoverLocalFiles()               — working → ready/error based on existsSync
  3. scanVaultForOrphans()             — import files not yet tracked
```

Entradas `failed` del store viejo se migran a `working` para que `scanVaultForOrphans` pueda verificar existencia y recuperarlas.

---

## Análisis de monetización

### La pregunta correcta: arquitectura local ≠ modelo de negocio

| Eje                       | Pregunta                                 |
| ------------------------- | ---------------------------------------- |
| **A — Verdad local**      | ¿Filesystem o DB es la fuente de verdad? |
| **B — Modelo de negocio** | ¿Por qué te pagan?                       |

Decidir A (filesystem-first) **no condiciona B**. Obsidian es radicalmente local-first y monetiza con Obsidian Sync + Obsidian Publish + licencia comercial.

### Por qué un artefacto pesado es buena historia de monetización

1. **El contenido es pesado.** Un vault de video/media pesa decenas a cientos de GB. DIY-sync de 200 GB es lento y caro. Eso le da valor real a una "Cloud" manejada.
2. **El sharing requiere backend.** Nadie graba/genera solo para archivar local — lo hace para _mandárselo a alguien_. Compartir = subir + URL + viewer en browser. Es inherentemente un servicio de nube.
3. **El mercado es grande.** Loom (grabación de pantalla cloud-first) fue adquirida por Atlassian en 2023 por ~$1B.

### El modelo que se diseña solo

```
FREE (local-first, el diferenciador)
├── Uso local ilimitado
├── Librería filesystem-first (tus archivos son tuyos)
└── Reproducción/uso local

PAID — "Cloud / Pro" (la capa de red)
├── Share links + streaming en browser
├── Transcoding
├── Viewer analytics
├── Links con password / expiración
├── Team workspaces
├── Transcripción + resumen AI
└── Retención larga / backup manejado
```

**Regla no-negociable:** el valor pago vive en la capa de red/colaboración/hosting, nunca en features locales.

### Costos reales (sin maquillar)

1. **Te saca el lock-in más barato.** El usuario se va cuando quiere. Tenés que retener por **valor**, no por cárcel.
2. **El sync bidireccional en la nube es más difícil** sobre filesystem-truth que sobre DB-truth. Pero el MVP de monetización **no necesita sync bidireccional** — necesita _upload-to-share_.
3. **Local-first es invisible para el usuario mainstream.** "Tus archivos son tuyos" es pitch para devs/privacy folks. Para el ICP correcto (técnico), es una fortaleza.

### El camino al primer dólar

```
Archivo local
   → botón "Share"
   → POST al backend (object storage, p. ej. R2)
   → backend devuelve URL de streaming
   → guardás { state, url, uploadedAt } en el SIDECAR
```

Eso es: un uploader + hosting/streaming + una página de viewer. **No es un motor de sync.** La parte difícil (sync multi-dispositivo) se difiere hasta tener ingresos que la justifiquen.

### Bring-your-own-storage (R2/S3)

El usuario apunta la app a su propio bucket R2/S3. Vos cobrás el software/la capa de streaming y ellos pagan su propio storage. Los costos de infra quedan cerca de cero.

- R2 tiene **cero egress** → nadie paga cuando alguien mira/descarga un archivo.
- Encaja perfectamente con un stack de Cloudflare Workers/R2.
