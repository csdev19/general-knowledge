# Patrón de Backlog

_Cómo mantener una carpeta `backlog/` que es la fuente de verdad de "lo que viene", sin perder nunca el trabajo diferido._

Un `backlog/` es una carpeta de documentación (no un tablero de tickets) donde vive todo el trabajo **planeado, propuesto o pausado que aún no está terminado**. Es el mapa de "lo que viene", con suficiente contexto en cada ítem para retomarlo sin re-investigar.

La idea central: cuando pospones una idea, una mejora o un análisis, **no se pierde**. Queda escrito con su contexto. Cuando lo retomás semanas después, no arrancás de cero.

## Anatomía

```
backlog/
├── index.md              # El mapa: tabla con todos los ítems y su estado
├── <feature-a>.md        # Un ítem = un archivo, con contexto para retomarlo
├── <feature-b>.md
└── ...
```

- **`index.md`** — la tabla-mapa. Fuente de verdad de "qué sigue".
- **Un ítem = un archivo** — cada ítem con sustancia gana su propio doc. Los ítems triviales (solo una idea ⚪) pueden vivir únicamente como fila del mapa, sin doc propio.

## Leyenda de estado

Emojis para escanear el estado de un vistazo:

| Emoji | Significado                                    |
| ----- | ---------------------------------------------- |
| 🟡    | En progreso (posiblemente pausado)             |
| 🟢    | Listo para validar (mergeado / en prod)        |
| 🔵    | Propuesto (analizado, aún no empezado)         |
| ⚪    | Idea (sin análisis todavía)                    |

Se puede extender con estados terminales (`✅ Decidido y cerrado`) o de seguimiento (`A vigilar`), pero estos cuatro son la base. Un ítem `✅` que ya está validado se **saca del mapa** y su aprendizaje se mueve al doc de feature/arquitectura correspondiente.

## Formato del mapa (tabla del index)

Cada fila es un ítem. Columnas mínimas: **Ítem · Área · Estado · Esfuerzo · Detalle (link)**.

```markdown
| Ítem                             | Área      | Estado                          | Esfuerzo   | Detalle                  |
| -------------------------------- | --------- | ------------------------------- | ---------- | ------------------------ |
| **Nombre del ítem** (contexto)   | Pipeline  | 🟢 Mergeado (#12) · validar     | Medio      | [Ver](./mi-item)         |
| Otro ítem más chico              | Editor    | 🔵 Propuesto                    | Bajo–Medio | [Ver](./otro-item)       |
| Una idea suelta                  | Infra     | ⚪ Idea                         | Alto       | —                        |
```

- **Área** agrupa por dominio/subsistema (Pipeline, Editor, Infra, QA, DX...).
- **Estado** incluye el emoji + una nota corta (número de PR, "validar en prod", etc.).
- **Esfuerzo** es una estimación gruesa (`Bajo` / `Medio` / `Alto` / `Variado` / `—`).
- **Detalle** enlaza al archivo del ítem, o `—` si no tiene doc propio.

## Formato de un ítem (un archivo)

Cada ítem-archivo debe tener **lo justo para retomarlo sin re-investigar**: qué es, en qué estado quedó, qué se decidió, qué falta.

```markdown
# <Nombre del ítem>

> **Estado: <emoji> <resumen de una línea>.** Dónde quedó, qué falta para cerrarlo.
> Enlaces al spec/plan si existen.

Uno o dos párrafos: qué problema resuelve y por qué importa.

## Qué se hizo / qué shipeó

| Pieza        | Qué hace                          |
| ------------ | --------------------------------- |
| ...          | ...                               |

## Qué falta / cómo validar

- [ ] Paso pendiente 1
- [ ] Paso pendiente 2

## Fuera de alcance (ítems separados)

- Cosa relacionada → enlace a su propio ítem del backlog.
```

Lo importante no es la plantilla exacta sino el principio: **el archivo tiene que dejar al próximo lector (o a vos en un mes) listo para actuar**, sin volver a investigar.

## Cómo se usa

- Cada ítem que se empieza a trabajar pasa a 🟡 y, si tiene sustancia, gana su propio doc.
- Al completar un ítem: mover el aprendizaje al doc de feature/arquitectura correspondiente y **quitarlo del mapa**.
- Mantener **esfuerzo** y **estado** siempre actualizados — es la fuente de verdad de "qué sigue".

## Por qué ayuda

- **Nunca se pierde trabajo diferido.** Toda idea o mejora pospuesta queda escrita con su contexto.
- **Se retoma sin costo.** El contexto vive en el ítem, no en la cabeza de quien lo pausó.
- **Se ve el estado de un vistazo.** El mapa con emojis dice qué sigue sin abrir nada.
- **Separa "pensar" de "hacer".** Un ítem 🔵 propuesto ya tiene el análisis; empezarlo es solo ejecutar.
