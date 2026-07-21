# Workflow de Specs y Plans

_El flujo brainstorm → spec (diseño) → plan (implementación) → archivar, y cómo se conecta con el backlog._

Dos carpetas hermanas sostienen todo el trabajo con sustancia:

```
docs/
├── specs/    # Documentos de diseño: QUÉ y POR QUÉ
└── plans/    # Planes de implementación: CÓMO, paso a paso
```

Separar diseño de implementación evita mezclar "qué queremos y por qué" con "en qué orden tocamos los archivos". El spec se puede discutir y aprobar antes de escribir una sola línea de plan.

## El flujo

```
brainstorm  →  spec (diseño)  →  plan (implementación)  →  ship  →  archivar
   (idea)       specs/YYYY-...     plans/YYYY-...          (PRs)   (mover aprendizaje)
```

1. **Brainstorm** — la idea nace, normalmente como ítem ⚪/🔵 en el [backlog](./backlog-pattern.md).
2. **Spec (diseño)** — cuando la idea vale la pena, se escribe un doc de diseño en `specs/`. Define objetivo, no-objetivos, arquitectura y decisiones. Es lo que se revisa y aprueba.
3. **Plan (implementación)** — a partir del spec aprobado, se escribe un plan en `plans/`: tareas concretas, orden, archivos a crear/tocar, checkboxes para trackear. Un spec grande puede partirse en varios planes numerados.
4. **Ship** — se ejecuta el plan tarea por tarea (código + tests + docs). Los checkboxes marcan el avance. **Para ejecutar con agentes en paralelo**, el plan primero se convierte en entregables de backlog autosuficientes y queda superseded — ver [plan → backlog](./plan-to-backlog.md).
5. **Archivar** — cuando shipeó: el aprendizaje duradero se mueve al doc de feature/arquitectura, el ítem sale del mapa del backlog, y el spec/plan quedan como registro histórico.

## Convención de nombres

Archivos con fecha al frente para orden cronológico natural:

```
specs/YYYY-MM-DD-<topic>-design.md      # p.ej. 2026-06-28-version-gate-design.md
plans/YYYY-MM-DD-<topic>.md             # p.ej. 2026-06-28-version-gate.md
```

- El **spec** lleva el sufijo `-design`.
- El **plan** comparte fecha y topic con su spec, sin el `-design`.
- Cuando un plan se parte en fases, se numeran: `...-video-editor-01-foundation.md`, `...-02-timeline.md`, etc.
- La fecha es la de creación del doc, no la de ship.

## Qué va en cada uno

### Spec (`specs/`) — diseño

- **Encabezado**: título, fecha, branch, scope (qué apps/paquetes toca).
- **Goal / Non-Goals**: qué se resuelve y, explícito, qué se deja afuera (diferido).
- **Architecture**: el enfoque, el flujo de datos, las decisiones clave y su porqué.
- **Files**: qué se crea y qué se modifica (a alto nivel).

Responde **QUÉ** y **POR QUÉ**. No entra en el orden táctico de la implementación.

### Plan (`plans/`) — implementación

- **Goal / Architecture / Tech Stack**: resumen ejecutable de una o dos líneas cada uno.
- **Referencia al spec**: enlace al doc de diseño que lo origina.
- **Convenciones a leer antes de empezar**: idioma del repo, reglas de commit, cómo correr tests, etc.
- **Estructura de archivos**: crear/modificar, explícito.
- **Tareas con checkboxes** (`- [ ]`): en orden, idealmente estilo TDD, para trackear avance y permitir ejecución agéntica.

Responde **CÓMO** y **EN QUÉ ORDEN**.

## Cómo se conecta con el backlog

- El **backlog** es el mapa de "lo que viene"; `specs/` y `plans/` son el trabajo detallado detrás de un ítem con sustancia.
- Un ítem del backlog que arranca enlaza a su spec y su plan (`Design: /specs/...` · `Plan: /plans/...`).
- Al shipear, el ítem se marca 🟢/✅ y sale del mapa; el spec/plan quedan archivados como registro.

## Por qué ayuda

- **Diseño antes que código**: el spec se aprueba sin comprometer implementación.
- **Ejecución trackeable**: los checkboxes del plan hacen visible el avance y permiten retomar (o delegar a un agente) sin perder el hilo.
- **Registro histórico**: fecha + topic dejan un rastro cronológico de decisiones.
- **Encaja con el backlog**: idea → spec → plan → ship es el ciclo de vida completo de cada ítem.
