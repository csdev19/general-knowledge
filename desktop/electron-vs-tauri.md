# Electron vs Tauri: Performance y Tamaño para una App de Captura Multimedia

*ADR — por qué una app de escritorio con pipeline multimedia pesado puede quedarse en Electron, de dónde viene el gap real de performance, y los gatillos para reconsiderar Tauri.*

> Análisis del tradeoff de framework para una app de escritorio con captura/encode multimedia.

## Veredicto

**No reescribas a Tauri ahora.** El gap de performance que sentís en una app de captura pesada **no viene del framework** — viene del pipeline de captura+encode. Eso lo arreglás en Electron, sin reescribir. Tauri es real pero sus ventajas (RAM idle, tamaño, batería) son secundarias para este tipo de app y no bloquean el objetivo de entregarla. El diferenciador es el diseño, y Electron ayuda ahí.

```
Lo que el usuario SIENTE en una app de captura = pipeline de captura + encode.
Lo que Tauri mejora = RAM idle + tamaño + arranque + batería.
Para una app de captura pesada, lo primero domina. Y lo primero se arregla en Electron.
```

---

## Los números reales

| Métrica                        | Electron    | Tauri          | Delta              |
| ------------------------------ | ----------- | -------------- | ------------------ |
| Bundle/installer (app trivial) | 80–200 MB   | 2–10 MB        | ~10–25x            |
| RAM idle (app trivial)         | ~120–300 MB | ~30–50 MB      | ~5x / 58–75% menos |
| Arranque                       | baseline    | ~4x más rápido | —                  |

**El detalle que invalida estos números para una app multimedia:**

Una app de captura de referencia hecha en Tauri pesa ~96 MB. Si la webview fuera lo que domina el tamaño, pesaría 5–10 MB. Pesa ~96 porque una app de captura carga FFmpeg, codecs, y librerías de captura nativas — ese payload es el mismo en Electron que en Tauri.

**Conclusión del tamaño:** el delta real para una app multimedia es ~2x (≈200 MB Electron vs ≈96 MB Tauri), no ~25x. La webview no es lo que pesa; el payload multimedia sí.

---

## De dónde viene la performance REAL en una app de captura

### 1. Captura (el más importante)

| Approach                                         | Qué es                                              | Costo                                                                  |
| ------------------------------------------------ | --------------------------------------------------- | ---------------------------------------------------------------------- |
| `getDisplayMedia()` + `MediaRecorder` (renderer) | API del browser, lo que Electron usa por defecto    | Funciona, pero corre en el renderer, poco control de bitrate/keyframes |
| **ScreenCaptureKit** (macOS 12.3+)               | API nativa moderna de Apple                         | Bajo. Eficiente.                                                       |

**Las apps rápidas lo son en gran parte porque capturan nativo, no porque Tauri sea mágico.**

### 2. Encode (donde se va la CPU)

| Approach                                                       | Costo CPU                                    |
| -------------------------------------------------------------- | -------------------------------------------- |
| Software (libx264 en CPU, típico de MediaRecorder)             | **Alto.** Esto es lo que calienta el laptop. |
| **Hardware** — VideoToolbox (macOS), NVENC/QuickSync (Windows) | **Bajo.** La GPU/chip hace el encode.        |

Este es el cuello de botella #1.

### 3. El framework (UI)

El render de la UI no es el cuello de botella durante la captura. La diferencia de Tauri está en **RAM idle** y **arranque**, no en lo que pasa mientras la operación pesada corre.

> **El insight central:** los dos ejes que más afectan la experiencia (captura + encode) **no dependen de Electron vs Tauri**. Dependen de cómo se implementa el pipeline. Se puede tener captura nativa + encode por hardware **dentro de Electron**.

---

## La jugada de mayor leverage (en Electron HOY)

### 1. Mover el encode a FFmpeg con hardware (subproceso)

```bash
# macOS — captura via avfoundation + encode por hardware (VideoToolbox)
ffmpeg -f avfoundation -framerate 60 -i "1:0" \
  -c:v h264_videotoolbox -b:v 8M -pix_fmt yuv420p out.mp4
```

**Si FFmpeg ya está bundleado, el costo de bundle es cero.** Solo hay que usarlo para capture+encode.

### 2. Single-window + matar renderers idle

Cada `BrowserWindow` es un Chromium aparte. Una sola ventana principal; destruir renderers que no estén en uso.

### 3. Trimming del bundle

Con `electron-builder`: exclusión de archivos, `files`/`!**/*.map`, remover locales de Chromium que no se usen.

---

## Por qué Electron AYUDA al diferenciador

Lo que diferencia a una app pulida: **performance de primera clase con diseño que da gusto usar**.

- Chromium completo = libertad total de CSS, animaciones, fuentes. Todo lo moderno funciona idéntico cross-platform.
- La WKWebView de Tauri en macOS tiene inconsistencias en CSS y render — fricción contra una UI pulida.

El framework que conviene para tener la UI más linda es, muchas veces, el que ya estás usando.

---

## El caso honesto a favor de Tauri (gatillos para reconsiderar)

Tauri tiene ventajas reales que podrían importar más adelante:

- **Batería / RAM idle:** ~40 MB vs ~180 MB en idle es real para una app que dejás corriendo todo el día.
- **"Se siente liviano/nativo":** complementa el objetivo de pulcritud.
- **Mobile (Tauri 2):** si algún día querés iOS/Android, Tauri 2 lo cubre.

**Gatillos para reconsiderar Tauri** (cualquiera de estos):

1. Conseguís tracción **y** el footprint/RAM idle se vuelve una queja _real y repetida_ de usuarios.
2. Querés mobile.
3. Electron optimizado **no** pasa el umbral de performance (ver abajo).

**El umbral sugerido:** "Si el Electron optimizado idlea bajo ~150 MB y hace la operación pesada (p. ej. graba 1080p60) bajo ~15% CPU con encode por hardware, el delta de Tauri no justifica reescribir."

---

## Secuencia recomendada

1. **Ahora:** entregá el Electron que existe (filesystem-first + lo que ya funciona).
2. **Optimización #1:** pipeline FFmpeg + encode por hardware. Es el 80% del "feel".
3. **Optimización #2:** single-window estricto, matar renderers idle, trimming de bundle.
4. **Medí** contra el umbral.
5. **Solo si** no pasás el umbral, o querés mobile, o el footprint se vuelve queja real → spike de Tauri.
