# Media Pipeline

*Un pipeline multimedia/compute pesado en Electron: captura → encode → escritura posicional a disco, con ventana de control flotante, estado cross-window, y recuperación de fallas.*

> **Framing.** Este doc describe un **pipeline de media/compute pesado** en una app de escritorio Electron — el caso concreto es captura de pantalla + encode de video, pero el patrón (engine fuera de React, escritura posicional a disco, estado cross-window vía un hub en el main, recuperación robusta de fallas) aplica a cualquier operación larga y costosa que produzca un archivo. Donde el texto dice "grabación", leelo como "la operación pesada".

El core real está construido sobre **mediabunny/WebCodecs** — MP4 H.264/AAC seekable, escrito a disco posicionalmente, con una **barra flotante de control**. La decisión mediabunny-sobre-FFmpeg se resume en [Por qué mediabunny y no FFmpeg](#por-qué-mediabunny-y-no-ffmpeg). Las primeras secciones (**Captura / Encoder / Data handling** legacy) describen un **stack de referencia** (`MediaRecorder`+FFmpeg), útil como contraste histórico, no el estado actual.

---

## Referencia histórica: el stack legacy

### TL;DR — legacy (referencia)

| Dimensión          | Legacy                                           |
| ------------------ | ------------------------------------------------ |
| Captura            | `getDisplayMedia()` en el renderer               |
| Encoder            | `MediaRecorder` VP9+Opus, 2.5 Mbps default       |
| Encode corre en    | Renderer (thread principal de React)             |
| Data handling      | Streaming a disco vía IPC (1 chunk/segundo)      |
| Output screen-only | WebM (VP9 + Opus)                                |
| Output con webcam  | MP4 (H.264 + AAC, re-encode FFmpeg post)         |
| FFmpeg bundleado   | ✅ Sí (binarios por plataforma)                  |

### Captura (legacy)

**Método:** `navigator.mediaDevices.getDisplayMedia()` en el renderer. Tres estrategias en cascada:

1. Audio con constraints explícitas (`echoCancellation: false`, 48kHz, stereo)
2. `audio: true` simple
3. Solo video como último fallback

El main process usa `desktopCapturer.getSources()` para listar pantallas.

### Encoder (legacy)

`MediaRecorder` con `video/webm;codecs=vp9,opus`, `videoBitsPerSecond: 2_500_000`, chunk interval 1000 ms. Webcam en paralelo con `vp9`, 1 Mbps. **El encode corre en el renderer** — compite con el thread de React.

### Data handling — streaming a disco (legacy)

**No hay acumulación en memoria.** Anti-OOM para operaciones largas:

```
ondataavailable → blob → base64 (renderer)
  → IPC append-chunk → main
  → WriteStream (backpressure, 1 MB buffer)
  → archivo temp en app.getPath('temp')
  → al parar: finalize → move → vault del usuario
```

### Post-procesamiento (legacy)

**Screen-only:** WebM directo → remux para agregar header de duración. **Con webcam:** MP4 via re-encode FFmpeg post (CRF 20, 4 threads, target 30fps). **Thumbnail:** PNG 320×240 extraído al segundo 1, en background.

---

## Arquitectura implementada (mediabunny/WebCodecs)

El core real está construido sobre **mediabunny/WebCodecs**, no `MediaRecorder`+FFmpeg. Sigue el modelo **Opción A**: el renderer de la ventana principal (oculta mientras la operación corre) captura y encodea; el main escribe a disco; una ventana flotante aparte muestra la barra de control.

```
Ventana principal (oculta)                  Barra flotante (ventana aparte)
 useScreenRecorder                           ControlBar  timer · 5 barras · ⏸/▶ · ■
  getUserMedia(chromeMediaSourceId) ─┐
  getUserMedia(mic) + sys audio ─────┤─► MediaStreamVideo/AudioTrackSource
  AudioContext mix + AnalyserNode    └─► Output(Mp4OutputFormat, StreamTarget)
                                          → chunks { data, position } ─┐
        ┌──────────────────────── main process ───────────────────────┘
        │ recording-writer  fd.write(buf, 0, len, position)  (posicional, no append)
        │ media-hub         relay estado↔comandos · oculta/muestra la principal
        │ finalize → vault <id>.mp4 + sidecar (título, duración, thumbnail)
        └──────────────────────────────────────────────────────────────
```

**Piezas:**

- `features/recording/recorder-store.ts` — **module-singleton** que posee el engine, la sesión del writer y el countdown 3-2-1. Deliberadamente vive fuera de React (mismo patrón que `ui/toast-store.ts`): sobrevive a navegación, remounts de página y desvíos al editor de screenshots. Ver [El engine ya no vive en la página](#el-engine-ya-no-vive-en-la-página) abajo.
- `features/recording/hooks/use-screen-recorder.ts` — binding fino de React sobre el store vía `useSyncExternalStore`; ya no tiene lógica propia.
- `features/recording/recorder-engine.ts` — captura + mezcla de audio + encode mediabunny (MP4 H.264/AAC). Seam `videoProvider`/`audioProvider` para meter cámara/watermark después sin tocar lo de abajo.
- `features/recording/elapsed.ts` + `audio-levels.ts` — lógica pura testeada (clock con pausa, RMS→5 barras).
- `features/control-bar/` — barra flotante con tokens/`cx()`/`data-*` propios.
- `main/recording/recording-writer.ts` — escritura **posicional** + finalize al vault. Drena los writes en vuelo antes de cerrar el handle (los chunks llegan fire-and-forget por IPC).
- `main/recording/control-bar-window.ts` — `BrowserWindow` frameless/transparent/always-on-top, abajo-centro de la pantalla.
- `main/recording/media-hub.ts` — relay de estado/comandos + oculta/muestra la ventana principal.

**IPC nuevo:** `recording:create|write|finalize|abort` (writer) · `recording:report-tick` → `control:tick` (estado → barra) · `control:command` → `recording:command` (botones → engine) · `recording:start|stop` (mostrar/ocultar ventanas).

**Vault:** agnóstico de extensión (`.mp4`/`.webm`), con sidecar de duración + thumbnail; `app-media://thumb/<id>` sirve el póster.

### El engine ya no vive en la página

**Problema (encontrado en auditoría pre-testers):** el engine, la sesión del writer y el countdown vivían en refs de React dentro de `useScreenRecorder`, instanciado por la página principal. Cualquier unmount de esa página — navegar, tomar un screenshot a mitad de operación, un rescate por atajo reabriendo la ventana y remontando la página — dejaba el engine huérfano: seguía corriendo (o el listener de comandos moría con el unmount), pero Stop/Pause dejaban de tener efecto porque apuntaban a una instancia de hook que ya no existía.

**Solución:** el engine + countdown ahora viven en `recorder-store.ts`, un **module-singleton** (mismo patrón que `ui/toast-store.ts`): funciones planas mutan estado de módulo y notifican suscriptores vía `useSyncExternalStore`. El listener `onRecordingCommand` (pause/resume/stop desde la barra o el atajo global) se registra **una sola vez, a nivel de módulo** — no dentro de un efecto de React atado al mount de una página — así que sobrevive a cualquier navegación. Con esto:

- Remontar la página principal ya no resetea nada: todas las instancias de `useScreenRecorder` en la misma ventana leen el mismo estado.
- El **stop de la barra/atajo global cancela un countdown en curso** en vez de ser ignorado — antes solo actuaba sobre un engine ya vivo.
- La navegación al terminar (`/library/:id`) se dispara desde una suscripción en `AppShell` (el layout que nunca se desmonta entre rutas), no desde un callback pasado por la página activa.
- El screenshot (hotkey, botón del tray, o desde la página de Screenshots) se **bloquea con un toast** si el store no está `idle` — cubre tanto una operación activa como el propio countdown/`starting`.
- La ventana principal se oculta (`recordingStart` IPC) **antes** de que el engine adquiera streams, no después — antes se veía la propia app en los primeros ~0.5-1s.
- `fastStart: false` (antes `"in-memory"`) — mediabunny escribe a disco progresivamente en vez de bufferear todo en RAM del renderer hasta el stop (eso hacía OOM-crashear operaciones largas y perder todo si la app moría antes del stop).

### Estado global de la operación (entre ventanas)

**Problema:** cada ventana (página principal, Capture Panel) tiene su propia instancia de `useScreenRecorder` — comparten _código, no estado_. Sin coordinación, el Capture Panel (otra ventana) no sabe que hay una operación activa y mostraría "Start", permitiendo arrancar una segunda operación → errores.

**Solución:** el `media-hub` (main, única fuente de verdad) mantiene `RecordingActivity { active, status }` y lo **transmite a todas las ventanas** (`recording:state`) en cada transición (start/stop/pause/resume), re-emitiendo solo cuando el status cambia — no en cada tick de 10/s. Cada ventana lee esto con **`useRecordingActivity()`**, que además **consulta `getRecordingState()` al montar** (por si una ventana se abre con la operación ya en curso). Con eso:

- **Página principal:** una **tarjeta de cabecera** con estado + **timer en vivo**, la fuente con badge **LOCKED**, y una fila **Pause / Stop** (pause/resume expuestos desde `useRecordingSetup`); los controles de setup quedan `inert`.
- **Capture Panel:** un **banner rojo con timer en vivo**, controles `inert`, y el botón pasa a **"Stop"** (comando stop al hub) — imposible arrancar una segunda operación desde el tray.

El timer en vivo de ambas ventanas sale de `RecordingActivity.elapsedSeconds`, que el hub actualiza desde los ticks del engine y re-emite ~1/s (no en cada tick de 10/s).

**Settings compartidos (misma idea, bidireccional).** La fuente, los toggles (mic/audio/cámara) y el micrófono seleccionado viven en el hub como `RecordingSettings` — **una sola fuente de verdad** para todas las ventanas. `useRecordingSettings` consulta al montar, se suscribe a cambios y **escribe optimista** vía `recording-settings:update`; el hub mergea, **re-emite a todas las ventanas** y de paso maneja la burbuja de cámara. Resultado: cambiás un toggle en la app o en el panel y **se refleja en ambos**.

### Cámara bubble (ventana flotante capturada)

La cámara es una **burbuja flotante** (`?window=camera-bubble`): una `BrowserWindow` circular, frameless/transparent/always-on-top, arrastrable, que muestra el webcam vía el hook testeado `useCameraPreview`. **No se compone en canvas** — como está en pantalla, la captura de pantalla la incluye en el video. La maneja el toggle Cámara (`camera:set` → `media-hub` → `CameraBubbleWindow.show/hide`).

Para que **la barra de control NO salga en el video** pero **la cámara sí**, la ventana de la barra usa `setContentProtection(true)` (macOS: `NSWindowSharingNone`); la burbuja deliberadamente no la usa. _(Verificar en runtime que el screen-capture respeta la exclusión.)_

**Se esconde durante un screenshot.** Como es always-on-top y no tiene content-protection (a propósito — para que la grabación la capture), quedaba dentro del selector nativo de región en cualquier captura mientras la cámara estaba prendida. `beforeCapture`/`afterCapture` llaman a `hub.hideCameraBubbleForCapture()` / `restoreCameraBubbleAfterCapture()`, que usan `hide()`/`show()` (no `destroy()`) — la cámara sigue tomada por `getUserMedia` todo el tiempo, así que no hay parpadeo del indicador de "en uso".

### Calidad configurable (presets + sliders)

La calidad es un **modelo puro** en `src/shared/recording-quality.ts` (sin imports de electron/node/react), compartido por main (validación) y renderer (UI + wiring):

- **Tres knobs** (`RecordingQuality { resolution, fps, bitrate }`) con pasos discretos. `resolution` 720/1080/1440/2160 → dimensiones 16:9; `fps` 24/30/48/60; `bitrate` light/medium/high/max → 4/8/16/24 Mbps. El **peso ≈ bitrate** (`mbPerMinute`).
- **Presets como vista derivada:** `activePreset(quality)` matchea contra combos nombrados (p. ej. Liviano 720/24/light, Equilibrado 1080/30/medium, Máxima 1440/60/high); si no matchea → `"custom"`. No se guarda `preset` aparte, así que **no hay desync** preset/valores: clickear un chip aplica el combo; mover cualquier slider cae en Personalizado.
- **Persistencia:** vive en `AppSettings.recordingQuality` (reusa el `settings-store`), validado por `sanitizeQuality` en `mergeSettings`.
- **Wiring al encoder:** `qualityToEngine(quality)` → `{ width, height, frameRate, videoBitrate }`. El engine usa esos valores en las constraints `maxWidth/maxHeight/maxFrameRate` y en el bitrate del track. **El `maxWidth/Height` es un techo: no upscalea** sobre la resolución nativa de la pantalla.
- **Una sola fuente para los números de encode:** el engine **no reescribe** ningún valor — su fallback es `qualityToEngine(DEFAULT_QUALITY)`, y el bitrate de audio sale de `AUDIO_BITRATE_BPS`.
- **Sin enums:** los sets (`RESOLUTION_STEPS`, `FPS_STEPS`, `BITRATE_STEPS`, `NAMED_PRESETS`, `PRESET_ORDER`) son arrays `as const` y el tipo se deriva con `(typeof X)[number]`.

### Watermark (free → paid)

Un watermark **quemado en el video** solo para usuarios free; pagar lo quita. La decisión está **centralizada en un solo hook** — el resto de la app (engine incluido) no sabe de planes ni flags.

- **`useWatermark()`** (`features/watermark/use-watermark.ts`) es el único seam. Compone tres inputs: `entitlement.isPaid`, `featureFlag` y un **override de dev**. La regla pura `resolveWatermarkEnabled({flagOn, isPaid, devForce})` = `flagOn && !paid`. Devuelve `{ enabled, config }`.
- **Toggle de dev que NUNCA llega a prod:** un switch en Settings, **visible solo en dev**, guarda el override en `localStorage`. Tanto la sección del toggle como la lectura están gateadas por `import.meta.env.DEV` (literal en build), así que se **eliminan del bundle de producción** — y un usuario no puede sacarse el watermark seteando localStorage a mano.
- **Draw-pass (canvas compositor)** `watermark-compositor.ts`: el screen track va a un `<canvas>` (frame + el asset re-teñido), y se re-captura con `canvas.captureStream(frameRate)`. Ese track va al encoder. **Costo cero cuando está apagado:** si `enabled === false` se encodea el screen track directo (sin canvas).
- **Wiring:** `useWatermark()` → `use-recording-setup` (ref) → `StartInput.watermark` → `startEngine`. `EngineOptions.watermark` decide compositar o no; `stop()` corta el draw loop.

### Recuperación de fallas (no quedar trabado)

Las rutas de error restauran **siempre** el estado de la app:

- **Falla al arrancar** (permiso denegado, `getDisplayMedia` falla): la ventana se oculta optimistamente **antes** de que el engine adquiera streams, así que el `catch` debe deshacer ese hide explícitamente — llama a `recordingStop()` (mostrar ventana + restaurar Dock + esconder barra) y aborta la sesión — y vuelve a `idle` (no queda en un estado `error` separado), así el botón Start es usable de inmediato.
- **Falla al finalizar:** `recordingStop()` se llama **siempre**, fuera del try/catch.
- **Falla a mitad de operación** (encoder error, o el track de pantalla termina porque el usuario corta el screen-share): el engine la expone vía `onError` (el `errorPromise` de las sources + el evento `ended` del track) → corre el mismo `stop()` robusto (finaliza-o-aborta + restaura todo). Un flag `stopping` evita doble-teardown si el usuario toca Stop y la falla disparan a la vez.
- **El renderer principal muere o la ventana se cierra a mitad de operación** (crash, OOM, o cierre): no llega `recordingStop` desde un renderer muerto, así que `media-hub.ts` expone `forceReset()` (reseteo best-effort de `activity` + Dock + barra) llamado desde `render-process-gone` en main. Cerrar la ventana mientras `activity.active` además dispara un diálogo de confirmación que corre el stop normal (con timeout de 5s) antes de dejar cerrar — así el archivo se finaliza con lo capturado hasta ese momento en vez de perderse.
- **Quit del tray a mitad de operación** (`before-quit`): mismo patrón — si `hub.isActive()`, `preventDefault` + confirmación, se manda el comando `stop`, y recién cuando llega `recordingStop` (o pasa el timeout de 5s) se hace `app.quit()`. Con el fastStart en disco (ya no en memoria) el archivo queda finalizado con lo capturado hasta ese momento.

### Gotcha: ocultar la ventana principal y el Dock/Cmd+Tab (macOS)

El hub **oculta la ventana principal mientras la operación corre** (`mainWindow.hide()`) para que solo se vea la barra flotante. En macOS, ocultar la única ventana puede **sacar la app del Dock y del switcher (Cmd+Tab)**, y `show()` no siempre la devuelve. Por eso, al **detener**, el hub **re-afirma la activation policy** (`applyDockPolicy()`) y trae la app al frente (`app.focus({ steal: true })`).

La visibilidad en el Dock/switcher es un **setting persistido** (`showInDock`): `regular` (con Dock + Cmd+Tab) vs `accessory` (solo barra de menú/tray). Vive en `infrastructure/settings-store.ts` y se togglea desde Settings.

### El hueco "starting" después del countdown

Tras el countdown `3·2·1`, el engine entra en estado **`"starting"`** mientras adquiere los streams y arma el encoder — ahí `isRecording` todavía es `false` y la ventana **aún no se ocultó**, así que el botón Start quedaba visible y clickeable ~0.5–1s (se podía disparar una segunda operación). Se cierra por tres lados:

- El **overlay del countdown sigue arriba durante `"starting"`** mostrando un spinner. Como el overlay es un backdrop que captura clicks, el botón no se puede tocar. No hay parpadeo: `clearCountdown()` y `setStatus("starting")` ocurren en el mismo tick.
- **Guard de re-entrada** en `useScreenRecorder.start`: chequea `engineRef || sessionRef`, y `sessionRef` se setea sincrónicamente, así que un segundo `start()` durante "starting" retorna de inmediato.
- El botón Start queda `disabled` durante el countdown y "starting" (defensa extra).

**Cancelable en todo momento.** `CountdownOverlay` tiene un botón "Cancel" y un handler de Escape, ambos wireados a `stopOrCancelRecording()` — la misma función que maneja el stop de la barra/atajo global. Una sola rama de código para las tres fases: durante `"counting"` cancela el intervalo; durante `"starting"` marca un flag que se consume apenas el engine resuelve; con un engine vivo, finaliza normalmente.

### Después de la operación: navegación + estado "Saving"

Al detener, `useScreenRecorder` muestra un breve estado **"Saving…"** en la barra mientras el encoder hace flush y el archivo se finaliza (metadata, thumbnail, nombre) — `recordingFinalize` solo resuelve cuando todo eso existe en el vault, así que la espera es real (con un mínimo de ~600ms para que no parpadee en clips cortos). Luego dispara `onComplete(recording)` que navega a `/library/:id` **antes** de volver a mostrar la ventana principal, para aterrizar directo en el detalle (sin flash de la página principal).

### Ventanas flotantes transparentes (barra y burbuja)

La barra y la burbuja son `BrowserWindow` frameless/transparent/always-on-top. Detalles que costaron y conviene recordar:

> **Dos causas de "caja oscura" sobre una ventana transparente** (ambas se ven parecidas, pero la raíz es distinta — confundirlas hace perder tiempo):
>
> 1. **Falta `backgroundColor: "#00000000"`** (junto con `transparent: true`): la ventana pinta un fondo opaco que se ve como una caja oscura sobre fondos claros. Obligatorio en **toda** ventana transparente.
> 2. **Un `box-shadow` de CSS más grande que el margen transparente de la ventana**: la sombra **se recorta contra los bordes (rectangulares) de la ventana** y queda una caja oscura. Regla: la sombra debe **desvanecerse por completo dentro del margen** (window − elemento). Ej. burbuja: window 224 − círculo 192 = 16px de margen, así que un blur de ~8px entra holgado; un blur de 18px se cortaba.

- **No debe scrollear.** El renderer marca `body[data-window="control-bar"]` y `main.css` la fija al viewport con `overflow: hidden` y le da a `#root` un _block-formatting context_, evitando que el margen superior del pill colapse hacia el body y empuje el documento (lo que sacaba una scrollbar).
- **Diseño:** rectángulo redondeado (no pill), botones cuadrados redondeados, orden `punto · timer · separador · medidor de mic · pausa · stop · atajo`, sin focus-ring, sombra discreta.

### Encode por hardware (VideoToolbox) y el caveat de OpenH264

mediabunny pasa `hardwareAcceleration: 'no-preference'` por defecto, y en la build de Chromium de Electron eso seleccionaba el **encoder H.264 por software (OpenH264)** — visible como warnings `[OpenH264] ... bitrate can't be controlled...` en consola, e implica más CPU/calor que el hardware. El engine ahora pide **`hardwareAcceleration: "prefer-hardware"`** (+ `latencyMode: "realtime"` para captura en vivo) en el `VideoEncodingConfig` del `MediaStreamVideoTrackSource`, para empujar a **VideoToolbox**. Es un _hint_: si el encode H.264 por hardware no está disponible vía WebCodecs, cae a software (sin romper).

**Verificación:** el `onEncoderConfig` loguea el config real; si el hardware engancha, los mensajes `[OpenH264]` **desaparecen** de la consola.

### Por qué mediabunny y no FFmpeg

No se portó el stack `MediaRecorder`+FFmpeg del legacy. Gran parte del rol de FFmpeg ahí era **parchear lo que `MediaRecorder` rompe** (WebM sin duration header → remux). mediabunny no usa `MediaRecorder`, así que esos parches no nacen.

| Aspecto          | Legacy                     | Implementado (mediabunny)                   |
| ---------------- | -------------------------- | ------------------------------------------- |
| Encoder          | `MediaRecorder` VP9/Opus   | WebCodecs H.264/AAC (HW, ~8× más rápido)    |
| Output           | WebM roto → remux FFmpeg   | MP4 seekable de un solo paso                |
| Hot path a disco | base64 → IPC → WriteStream | bytes crudos posicionados (`StreamTarget`)  |
| Compositing      | FFmpeg `filter_complex`    | canvas + `CanvasSource` en vivo (follow-up) |
| Bundle           | ~50–80MB binarios por arch | mediabunny tree-shakable, sin deps          |

### Pendiente (follow-ups diseñados)

- **Cámara PiP** — compositar el webcam vía `CanvasSource` en el seam `videoProvider`.
- **Watermark** — pasada de dibujo extra en el mismo compositor.
- **Arrancar desde el Capture Panel** — hoy su botón abre la ventana principal (el encode no debe correr en la ventana transparente del menu-bar); falta el IPC panel→main para arrancar.
