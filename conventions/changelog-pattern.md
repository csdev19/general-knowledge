# Patrón de Changelog en el docs app (diario de decisiones)

_Cómo mantener un changelog narrativo en la app de documentación sin que el mantenimiento del índice se coma el valor: un archivo por entrada, índice auto-generado, y un listón claro de qué merece entrada._

El changelog del docs app **no es un log de commits** — eso ya lo genera release-please automáticamente por app (`CHANGELOG.md` desde Conventional Commits; ver [release-automation](../monorepos/release-automation.md)). El changelog del docs app es el **diario de decisiones**: qué se hizo, por qué, qué se descartó, y qué lección dejó. Es la fuente para recuperar contexto entre sesiones (propio o de agentes) después del squash-merge, cuando la historia del PR ya se aplanó.

## El problema que evita este patrón

La versión ingenua (probada en trip-planner durante ~6 meses, 46 entradas) mantenía un `index.mdx` a mano con una card + una fila de tabla por entrada. Fallas acumuladas:

1. **Cada entrada se escribía 3 veces** — el archivo, la card del índice, la fila de la tabla. Las descripciones de las cards degeneraron en párrafos que duplicaban la entrada completa (drift garantizado).
2. **El índice era un imán de merge conflicts** — todo PR tocaba el mismo archivo en el mismo lugar (tope de la lista). Con agentes trabajando en paralelo, choque seguro.
3. **Template con tablas de commit hashes** — el hash real no existe hasta el squash-merge, así que las tablas nacían inventadas. Release-please ya mapea commits a versiones.
4. **Sin listón de entrada** — fixes triviales recibían la misma ceremonia que postmortems, ahogando la señal.

## Anatomía

```
apps/docs/src/content/docs/changelog/
├── index.mdx                       # Página índice: SOLO intro + <ChangelogIndex />
├── YYYY-MM-DD-titulo-corto.mdx     # Una entrada = un archivo
└── ...
src/components/ChangelogIndex.astro # Genera las cards desde la colección
```

- **Una entrada = un archivo**, nombrado `YYYY-MM-DD-titulo-kebab.mdx`. El prefijo de fecha es obligatorio: es la clave de ordenamiento del índice.
- **El índice se auto-genera** desde la content collection. Agregar una entrada = crear el archivo. Nadie edita `index.mdx` nunca → cero doble escritura, cero conflicts.

## El componente índice (Astro Starlight)

```astro
---
// src/components/ChangelogIndex.astro
// El sort por id ES el sort cronológico inverso, gracias al prefijo YYYY-MM-DD
// del nombre de archivo — sin parsear fechas.
import { getCollection } from "astro:content";
import { CardGrid, LinkCard } from "@astrojs/starlight/components";

const entries = (await getCollection("docs"))
  .filter((entry) => entry.id.startsWith("changelog/"))
  .sort((a, b) => b.id.localeCompare(a.id));
---

<CardGrid>
  {
    entries.map((entry) => (
      <LinkCard
        title={entry.data.title}
        href={`/${entry.id}`}
        description={entry.data.description}
      />
    ))
  }
</CardGrid>
```

Y el `index.mdx` queda reducido a frontmatter + un párrafo de intro + `<ChangelogIndex />`.

> En otros generadores (fumadocs, Docusaurus, VitePress) el mecanismo cambia pero el patrón es el mismo: el índice se deriva de los archivos, nunca se mantiene a mano.

## Formato de entrada

```mdx
---
title: "Month DD, YYYY - Short Title"
description: Resumen de 1-2 líneas — es el texto de la card del índice, corto
date: YYYY-MM-DD
tags:
  - changelog
  - relevant-tags
---

Párrafo de introducción.

---

## type: Change Title

### Changes
- Qué cambió, concreto

### Files Changed
paths/tocados

### Decision Rationale
Por qué, trade-offs, alternativas descartadas.
```

Reglas duras:

- **`description` de 1-2 líneas.** La narrativa vive en el cuerpo de la entrada; la card del índice solo anuncia. Si la description necesita un párrafo, el título está mal o la entrada cubre demasiado.
- **Sin tablas de commit hashes.** No existen hasta el merge, y release-please ya cubre esa capa.
- **`Decision Rationale` es la sección obligatoria** — es la única información que no vive en ningún otro lado (ni git log, ni el diff, ni release-please).

## El listón: qué merece entrada

Entrada **sí** (hay decisión o lección que registrar):

- Features nuevas, cambios de arquitectura, breaking changes
- Cambios de schema — especialmente migraciones y su historia en prod
- Fixes con causa raíz no obvia o con lección
- **Postmortems** — algo salió mal en deploy/prod y cómo se recuperó (las entradas más valiosas del patrón)

Entrada **no** (release-please ya lo registra):

- Fixes triviales, typos, bumps de dependencias, refactors mecánicos sin historia

## Relación con release-please

Dos capas complementarias, sin solaparse:

| Capa                | Quién la escribe             | Qué cuenta                          |
| ------------------- | ---------------------------- | ----------------------------------- |
| `CHANGELOG.md` por app | release-please (automático) | Qué commits entraron en qué versión |
| `docs/changelog/`   | el autor o el agente (manual, con listón) | Por qué, qué se descartó, lecciones |

Si un cambio no tiene "por qué" que contar, no duplica capa: vive solo en la de release-please.
