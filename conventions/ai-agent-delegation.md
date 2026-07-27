# Delegación a agentes: los dos modos

_Cuándo usar un modelo caro y rápido, cuándo usar modelos baratos de larga ejecución, y — más importante — por qué el tier del modelo casi nunca es lo que hace lento un trabajo delegado._

Este doc asume una configuración con un agente orquestador que despacha subagentes
(Claude Code con `Agent`/`Task`, o equivalente). Está escrito para que una IA sin
contexto previo pueda implementar la delegación leyendo solo esto.

## Los dos modos

| | **Modo A — rápido, medianamente caro** | **Modo B — lento, barato** |
| --- | --- | --- |
| Modelos | Opus 4.8 | Sonnet + Haiku |
| Cuándo | Estás esperando el resultado, sesión interactiva | No estás mirando: corre en background, en una máquina siempre encendida (p. ej. un Mac mini) |
| Criterio de elección | El costo del tiempo de espera humano supera el costo de tokens | El trabajo puede tardar; nadie está bloqueado |
| Riesgo | Gasto | Que un modelo barato se atasque y nadie lo note → hace falta un heartbeat o un review final |

La regla operativa: **el modo lo elige quién está esperando, no la dificultad de la
tarea.** Una tarea difícil que nadie está mirando va en modo B con un review final
caro; una tarea trivial que bloquea a una persona va en modo A.

## Reparto de roles dentro de una ejecución

Cualquiera de los dos modos usa la misma forma. Lo que cambia es qué modelo ocupa
cada casilla.

| Rol | Qué hace | Modo A | Modo B |
| --- | --- | --- | --- |
| **Orquestador** | Lee el plan, despacha, revisa entre tareas, mantiene el ledger | Opus 4.8 | Opus 4.8 (el orquestador nunca baja de tier: es quien decide) |
| **Implementador mecánico** | El plan ya trae el código/YAML/JSON completo → transcribir + validar | Haiku | Haiku |
| **Implementador con juicio** | Lógica real, verificar afirmaciones contra archivos, formatos delicados | Opus 4.8 | Sonnet |
| **Reviewer por tarea** | Un diff acotado contra su brief | Sonnet | Sonnet |
| **Review final de rama** | Decide si algo rompe producción | Opus 4.8 | Opus 4.8 (**nunca** se abarata) |

Las dos casillas que no se abaratan nunca son el **orquestador** y el **review final
de rama**. Todo lo demás sí.

## El tier del modelo no es el cuello de botella

Datos medidos en una sesión real: 21 subagentes, 10 tareas de configuración de CI
(YAML + JSON + docs), plan con el código completo inline.

| Implementadores | Duración media | Tool calls por tarea |
| --- | --- | --- |
| Haiku (6 tareas) | 94 s | 10–17 |
| Sonnet (3 tareas) | 75 s | 10–11 |

Haiku dio más vueltas — el efecto conocido de "el modelo barato usa más turnos" —
pero fue leve. **Pasar las 6 tareas mecánicas de Haiku a Sonnet habría ahorrado unos
2 minutos sobre 41.**

Dónde se fue el tiempo de verdad:

```
Tiempo total de subagentes ≈ 41 min

  Review final de rama (1 agente Opus) ....... 11 min   ← el más lento de todos
  9 reviewers por tarea (Sonnet) ............. 13 min
  9 implementadores (Haiku + Sonnet) ......... 13 min
  Fixers ......................................4 min
```

Conclusión, y es lo único que hay que recordar de este doc: **el review final caro
tardó más que los seis implementadores baratos juntos.** La lentitud vino de la
_arquitectura del proceso_ — 18 despachos secuenciales — no del tier de los modelos.

⚠️ Muestra de una sesión (n=21) sobre trabajo de configuración. No lo tomes como ley:
en tareas con más razonamiento la brecha entre tiers se ensancha. Lo que sí generaliza
es el orden de magnitud: **la estructura domina al tier.**

## Las reglas que sí recortan tiempo

En orden de impacto medido:

1. **Agrupa tareas hermanas en un solo despacho.** En la sesión medida, cuatro tareas
   producían cuatro archivos YAML casi idénticos y se despacharon por separado: 8
   despachos (4 implementadores + 4 reviewers) donde bastaban 2. Costo del error:
   ~10 minutos, el ahorro más grande disponible. **Si dos tareas consecutivas tocan
   archivos del mismo tipo con la misma forma, son una sola tarea.**
2. **Corre los reviewers en background y en paralelo.** Un reviewer no bloquea el
   siguiente implementador: solo lee un diff ya commiteado. Secuenciarlos es tiempo
   muerto puro.
3. **Salta el review por tarea cuando la tarea es pura transcripción**, y apóyate en
   el review final de rama + el gate de CI. Ver la sección siguiente para cuándo NO
   saltarlo.
4. **No hagas un fixer por hallazgo.** Cuando el review final devuelve N hallazgos,
   despacha **un** agente con la lista completa. N fixers reconstruyen contexto N
   veces y re-corren suites N veces.

## Cuándo el review por tarea sí se paga

Saltarlo no es gratis. En la sesión medida, un reviewer Sonnet de 81 segundos
encontró un bug **Critical** que el orquestador había escrito en el plan: un PR gate
que usaba `git diff BASE HEAD` (dos puntos) en vez de `BASE...HEAD` (tres puntos).
Con dos puntos, si la rama base avanza después del fork, el diff reporta archivos que
el PR nunca tocó — y el gate pasa exactamente cuando debía fallar.

Heurística:

- **Sí review por tarea** si la tarea contiene lógica ejecutable (shell, condiciones,
  cálculos), afirmaciones que hay que verificar contra otros archivos, o algo cuyo
  fallo es silencioso.
- **No hace falta** si la tarea es escribir un archivo cuyo contenido completo ya
  venía en el brief y hay un validador automático (linter, type-check, `actionlint`).
  El review final de rama lo cubre.

Un review barato que encuentra un fallo silencioso se paga solo. Un review barato
sobre una transcripción ya validada por un linter, no.

## Cómo se despacha (para que el contexto no explote)

Todo lo que pegues en un prompt de despacho — y todo lo que el subagente imprima de
vuelta — se queda en el contexto del orquestador para el resto de la sesión. Pasa los
artefactos **como archivos**, no como texto pegado.

```
.<scratch>/task-N-brief.md     extraído del plan; es la ÚNICA fuente de requisitos
.<scratch>/task-N-report.md    el subagente escribe aquí; devuelve solo un resumen
.<scratch>/review-<a>..<b>.diff  commits + stat + diff -U10 del rango, en un archivo
```

Un prompt de despacho contiene, y nada más:

1. Una línea de dónde encaja esta tarea en el proyecto.
2. La ruta del brief, presentada como "leelo primero, son tus requisitos, con los
   valores exactos a usar literalmente".
3. Interfaces y decisiones de tareas anteriores que el brief no puede conocer.
4. Tu resolución de cualquier ambigüedad que hayas notado en el brief.
5. La ruta del archivo de reporte y qué debe contener.

**Nunca** pegues el historial acumulado de tareas previas ("estado después de las
tareas 1-3") en un despacho posterior. Un subagente fresco necesita su tarea, sus
interfaces y las restricciones globales. Nada más.

### Reglas al escribir el prompt de un reviewer

- Copia las restricciones vinculantes **literalmente** del plan: valores exactos,
  formatos exactos, relaciones declaradas entre componentes.
- **No pre-juzgues hallazgos.** Nunca escribas "no marques X", "trátalo como Minor a
  lo sumo", ni "el plan eligió esto". Si crees que un hallazgo sería un falso
  positivo, deja que el reviewer lo levante y resuélvelo tú después. Si el prompt que
  estás escribiendo contiene "no marques", párate: estás ahorrándote una ronda de
  review a costa del review.
- Dile lo que **no** necesita re-correr (tests que el implementador ya corrió) y lo
  que **sí** debe verificar de forma independiente contra archivos reales.

## Progreso durable: el ledger

La memoria de conversación no sobrevive a una compactación. El fallo más caro
observado en la práctica es un orquestador que perdió el hilo y **re-despachó
tareas ya completas**.

Mantén un archivo de progreso (p. ej. `.<scratch>/progress.md`), fuera de git o
git-ignored, y al cerrar cada tarea añade una línea:

```
Task N: complete (commits <base7>..<head7>, review clean)
```

Reglas:

- Al arrancar, lee el ledger. Lo que figure como completo **está** completo: no lo
  re-despaches, retoma en la primera tarea sin marcar.
- Después de una compactación, confía en el ledger y en `git log` antes que en tu
  propio recuerdo. Los commits que el ledger nombra existen en git aunque tú ya no
  recuerdes haberlos creado.
- Anota también los hallazgos **Minor** que decidiste no arreglar en el momento, y
  apunta el review final a esa lista. Un roll-up que nadie lee es un descarte
  silencioso.

## Lección transversal: nombra la unidad antes de citar el número

Esto salió de la misma sesión y no es sobre agentes, es sobre medir. Costó dos
correcciones y una decisión tomada dos veces.

Se necesitaba saber con qué frecuencia ocurría cierta forma de commit para decidir si
valía la pena un guard automático. Tres intentos:

1. **"0 ocurrencias en 674 commits"** — se muestreó `git log -200`. Pero el repo
   commiteaba ~200 veces al día, así que esa ventana cubría **un solo día** y se
   generalizó a toda la historia. → **`git log -N` no es una ventana de tiempo.**
   Con tasa de commits desigual, N commits no dice nada sobre un rango de fechas.
2. **"9 ocurrencias, 8 en 11 días"** — ventana corregida (los 674 commits), pero se
   contaron **commits** cuando el mecanismo evaluado era un **PR check**. El número
   invirtió la recomendación midiendo la cosa equivocada.
3. **Correcto** — se contaron merges a la rama principal: **2 ocurrencias en toda la
   historia del repo.** Los 9 commits del intento 2 eran commits internos de PRs que
   sí tocaban a los consumidores; nunca hubo nada roto.

Las dos reglas que quedan:

- **Declara el rango de fechas que tu muestra cubre realmente** antes de generalizar
  desde ella.
- **Haz coincidir la unidad de medida con el mecanismo que evalúas.** Un guard a
  nivel de PR se mide por PR, no por commit. Un guard a nivel de commit, por commit.

Corolario para agentes: cuando presentes una medición como justificación de una
decisión, incluye el comando exacto que la produjo. Fue leyendo el comando que se
detectaron los dos errores.

## Checklist de arranque

Para implementar esta delegación en un proyecto nuevo:

- [ ] Definir qué modelo cubre el modo A y qué modelos el modo B, y anotarlo donde el
      orquestador lo lea (`CLAUDE.md`, `AGENTS.md` o equivalente).
- [ ] Fijar orquestador y review final de rama en el modelo caro. No negociable.
- [ ] Crear el directorio de scratch para briefs, reportes y diffs; git-ignorarlo.
- [ ] Crear el ledger de progreso y la regla de leerlo al arrancar.
- [ ] Antes de despachar la tarea 1: revisar el plan buscando tareas hermanas y
      agruparlas. Es el ahorro más grande y solo está disponible antes de empezar.
- [ ] Decidir, tarea por tarea, si lleva review propio (lógica ejecutable / fallo
      silencioso → sí; transcripción con validador → no).
- [ ] Correr el gate completo del proyecto (type-check, lint/format, tests, build)
      **una vez, antes de push**, no entre tareas.

## Ver también

- [specs-and-plans-workflow.md](./specs-and-plans-workflow.md) — de dónde sale el plan
  que esta delegación ejecuta.
- [plan-to-backlog.md](./plan-to-backlog.md) — convertir un plan aprobado en
  entregables autosuficientes para agentes en paralelo.
