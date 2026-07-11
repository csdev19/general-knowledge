# Analytics & Feature Flags

*Arquitectura de analytics en una app de escritorio Electron — renderer-primary + main-sink, feature flags offline-safe, identidad por dispositivo, regla de dos canales de error, y postura de privacidad.*

Documenta la capa de analytics (aquí, PostHog) y feature flags del desktop.

---

## Arquitectura: renderer-primary + main-sink

El SDK de analytics no corre en un solo lugar — se divide entre el renderer de la ventana principal y el proceso main, cada uno responsable de lo que puede observar.

### Ventana principal (renderer)

`posthog-js` se inicializa en la ventana principal (`src/renderer/src/features/analytics/posthog.ts`). Es el canal primario porque:

- Tiene acceso a `posthog-js`, que soporta feature flags y exception autocapture del renderer.
- Puede llamar a `posthog.identify(deviceId)` para asociar eventos a la instalación.
- Puede evaluar flags directamente en React con el hook `useFlag`.

### Proceso main (sink)

`posthog-node` corre en el proceso main (`src/main/services/analytics.service.ts`) como segundo canal. Sus dos roles:

1. **Crashes de Node:** captura `uncaughtException` y `unhandledRejection` del proceso main, que el renderer nunca vería.
2. **Sink IPC:** las ventanas secundarias (control-bar, camera-bubble, capture-panel) no inicializan su propio SDK. En cambio, llaman a `installCrashForwarder(origin)`, que intercepta errores del renderer y los reenvía al proceso main vía IPC `analytics:capture-exception`. El handler en `analytics-ipc.ts` los envía con `posthog-node`.

```
Ventana principal          Ventanas secundarias
  posthog-js               control-bar · camera-bubble · capture-panel
  └─ identify(deviceId)      └─ installCrashForwarder(origin)
  └─ exception autocapture       └─ IPC analytics:capture-exception
  └─ useFlag(name)                    │
                                      ▼
                           Proceso main
                             posthog-node
                             └─ uncaughtException / unhandledRejection
                             └─ sink de errores de ventanas secundarias
```

---

## Feature flags

### Hook `useFlag(name)`

`useFlag` (`src/renderer/src/features/analytics/use-flag.ts`) es el único punto de acceso a flags en el renderer. Internamente consulta `posthog.getFeatureFlag(name)` y devuelve el valor, o el default si el SDK no está listo, no hay key configurada, o no hay red. La app nunca lanza o se degrada por un flag indefinido.

### Flags definidos (ejemplos)

| Nombre              | Tipo    | Default | Propósito                                                                                                                            |
| ------------------- | ------- | ------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| `watermark-enabled` | boolean | `true`  | Kill-switch remoto del watermark. Default `true` → el watermark sale en prod aunque no haya red. El hook de watermark lo lee directo. |
| `bypass-login`      | boolean | `true`  | Seam para la futura pantalla de login. No existe UI de login aún; el default lo trata como bypassed.                                 |

### Comportamiento offline

Si la key de analytics no está configurada, o si el backend no responde, los SDKs entran en modo no-op: no envían nada y `useFlag` devuelve el default de cada flag. La app funciona exactamente igual que sin analytics.

---

## Identidad por dispositivo

Al primer arranque, el settings-store acuña un `deviceId` (UUID v4) y lo persiste en `AppSettings`. Ese UUID es estable por instalación (no por usuario — no hay login aún).

La ventana principal llama a `posthog.identify(deviceId)` al iniciar. Dos super-propiedades se añaden a cada evento y error:

| Propiedad | Valor         | Uso                                                            |
| --------- | ------------- | -------------------------------------------------------------- |
| `product` | `<app-id>`    | Filtra eventos de este producto en dashboards multi-superficie |
| `surface` | `desktop`     | Distingue desktop de futuras superficies (web, mobile)         |

---

## Regla de dos canales de error

`reportError(userMessage, error, { context, retry })` implementa una regla estricta: el canal de usuario y el canal de desarrollador **nunca se mezclan**.

| Canal         | Qué recibe                                   | Cómo                                           |
| ------------- | -------------------------------------------- | ---------------------------------------------- |
| Usuario       | Mensaje legible, orientado a acción          | Toast visible con opción Reintentar            |
| Desarrollador | Stack completo, contexto técnico, origen     | `posthog.captureException(error, { context })` |

El toast muestra solo el `userMessage` — nunca el stack, nunca el nombre de clase del error. El backend de analytics recibe solo el payload técnico — nunca el texto que ve el usuario.

Los puntos de falla de la operación pesada usan esta función:

- Falla al iniciar (`start`)
- Falla a mitad de operación (`mid-recording`)
- Falla al finalizar (`finalize`)

Cada uno muestra un toast con **Reintentar** como acción segura (arrancar es 100% local, sin llamadas a API).

---

## Capa defensiva: ErrorBoundary + toast

### ErrorBoundary

Un React `ErrorBoundary` envuelve el árbol de la aplicación. Si un componente lanza una excepción no capturada:

1. Muestra un fallback amigable en lugar de pantalla en blanco.
2. Reporta la excepción al backend de analytics vía `posthog.captureException`.

El usuario ve un mensaje de error genérico, no una traza de stack.

### Primitivo `ui/toast`

`src/renderer/src/ui/toast` es el primitivo base para notificaciones en pantalla. Todas las llamadas a `reportError` pasan por acá — ninguna pantalla de error usa `alert()` o `console.error` como canal visible para el usuario.

---

## Privacidad

El foco en la privacidad no es opcional para una app que puede capturar la pantalla del usuario. La configuración de `posthog-js` desactiva explícitamente los canales invasivos:

| Capacidad               | Estado         | Motivo                                                                       |
| ----------------------- | -------------- | ---------------------------------------------------------------------------- |
| Session replay          | ❌ Desactivado | Grabaría la pantalla del usuario dentro de la app de captura — inaceptable   |
| DOM autocapture         | ❌ Desactivado | Captura texto e interacciones sin contexto — privacidad y ruido              |
| Click autocapture       | ❌ Desactivado | Idem                                                                         |
| Exception autocapture   | ✅ Activo      | Solo stacks de error — sin PII                                               |
| Feature flag evaluation | ✅ Activo      | Necesario para el comportamiento correcto de flags                           |
| Eventos explícitos      | ✅ Activo      | Solo los que el código llama explícitamente                                  |

---

## Pendientes (fuera de scope, documentados)

Los siguientes ítems están diseñados pero no construidos:

- **Login UI y entitlement real:** `isPaid` en el hook de watermark es un stub. El swap es una línea dentro del hook una vez que exista un plans/entitlement API. El seam ya está — nada más cambia.
- **Toggle de opt-out de telemetría en Settings:** agregar un toggle en Settings que llame a `posthog.optOut()` / `posthog.optIn()` y persista la preferencia en `AppSettings`.
- **Eventos de producto explícitos:** la infraestructura está lista pero solo captura errores y flags hoy. Eventos como `recording_started`, `recording_completed`, o `settings_changed` se pueden agregar cuando haya necesidad analítica real.
