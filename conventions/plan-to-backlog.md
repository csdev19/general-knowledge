# Plan → Backlog: entregables ejecutables en paralelo

_Cómo convertir un plan de implementación aprobado en documentos de backlog autosuficientes que varios agentes runner pueden ejecutar en paralelo — y por qué el backlog, no el plan, es la fuente de ejecución._

Extiende el [workflow specs + plans](./specs-and-plans-workflow.md): entre **plan** y
**ship** se inserta una conversión. El plan (un solo documento lineal) se transforma en
una **épica** + **un doc por entregable** dentro del `backlog/` de la app de
documentación, siguiendo el [patrón backlog](./backlog-pattern.md).

```
brainstorm → spec → plan → [CONVERSIÓN] → backlog épica + P1..PN → runners en paralelo → ship
                              plan queda superseded ──────┘
```

## Por qué (no perder esto de vista)

1. **Un plan es lineal; la ejecución no tiene por qué serlo.** El plan ordena tareas para
   un solo ejecutor. Al convertirlo, las dependencias reales se derivan de los bloques
   **Interfaces** (qué consume cada tarea), no del diagrama de carriles — que suele
   sobre-serializar. Resultado típico: 3 carriles paralelos donde el plan decía "serial".
2. **Un runner solo ve su doc.** Los subagentes ejecutan con contexto fresco y limitado.
   Por eso cada doc de entregable es **autosuficiente**: código completo copiado del plan
   (nunca "ver el plan"), precondiciones verificables con regla STOP-y-reporta, comandos
   con salida esperada, commit y Definition of Done. Si un doc requiere abrir otro
   documento para ejecutarse, la conversión falló.
3. **Una sola fuente de ejecución.** Tras la conversión, el plan se marca
   **superseded — do not execute from there**. Ejecutar desde dos documentos que pueden
   divergir es cómo se pierden pasos. El estado vive en el mapa del backlog.
4. **Paralelismo solo por archivos disjuntos.** Dos entregables corren en paralelo
   únicamente si sus conjuntos de archivos no se tocan. "Conceptualmente separados" no
   justifica paralelismo; colisión de archivos = serial.

## La skill que lo automatiza

Este patrón vive como skill de proyecto en `.claude/skills/plan-to-backlog/SKILL.md`
(en `monorepo-template` y en los proyectos derivados). La skill fija el inventario de
salida (épica + P-docs + filas del índice), las reglas de conversión pre-decididas y la
verificación (build de la app de docs antes de commitear).

**Evidencia de que vale la pena** (test TDD de la skill, 2026-07-21, language-cards):
un agente *sin* la skill logró la conversión correcta — el patrón es descubrible desde
los ejemplares del repo — pero gastó ~100k tokens y 30 tool uses en redescubrir
convenciones y tomó decisiones de juicio sobre la marcha. Con la skill: cumplimiento
estructural con 1 tool use y ~⅓ de los tokens, y las decisiones de juicio quedaron
fijadas. La skill no corrige errores: **elimina redescubrimiento y varianza**.

## Decisiones pre-tomadas (no re-litigar en cada conversión)

| Decisión | Regla |
| --- | --- |
| Idioma | Prosa en inglés; tokens de estado/esfuerzo en el idioma de la leyenda del índice (`🔵 Propuesto`, `Bajo/Medio/Alto`) |
| Dependencias | Derivadas de los bloques **Interfaces** del plan, no del diagrama de carriles |
| Paralelismo | Solo por conjuntos de archivos disjuntos, justificado explícitamente en la épica |
| Ramas | Integración nombrada en el plan; runs paralelos en `feat/<épica>-p<N>` que mergean de vuelta |
| Estados | 🔵 Propuesto → 🟡 En progreso → 🟢 Hecho; la épica pasa a 🟢 Listo para validar cuando todos los P-docs están 🟢 |
| Índice | Filas insertadas aditivamente; los estados de otras épicas los actualiza su propio trabajo |

## Ejemplares reales

- `language-cards` → `apps/documentation/src/content/docs/backlog/`: épicas
  `furigana-tokens` (E1–E7, ejecutada) y `particle-sound-tap-explain` (P1–P9) — la
  segunda, generada con la skill.
