# Changelog Pattern in the docs app (decision journal)

_How to keep a narrative changelog in the documentation app without index maintenance eating the value: one file per entry, an auto-generated index, and a clear bar for what deserves an entry._

The docs app changelog **is not a commit log** — release-please already generates that automatically per app (`CHANGELOG.md` from Conventional Commits; see [release-automation](../monorepos/release-automation.md)). The docs app changelog is the **decision journal**: what was done, why, what was rejected, and what lesson it left behind. It is the source for recovering context between sessions (yours or an agent's) after a squash-merge, once the PR history has been flattened.

## The problem this pattern avoids

The naive version (tried in trip-planner for ~6 months, 46 entries) maintained an `index.mdx` by hand with a card + a table row per entry. Accumulated failures:

1. **Every entry was written 3 times** — the file, the index card, the table row. The card descriptions degenerated into paragraphs duplicating the whole entry (guaranteed drift).
2. **The index was a merge-conflict magnet** — every PR touched the same file in the same place (top of the list). With agents working in parallel, a collision is certain.
3. **A template with commit-hash tables** — the real hash does not exist until the squash-merge, so the tables were born invented. Release-please already maps commits to versions.
4. **No entry bar** — trivial fixes got the same ceremony as postmortems, drowning the signal.

## Anatomy

```
apps/docs/src/content/docs/changelog/
├── index.mdx                       # Index page: ONLY intro + <ChangelogIndex />
├── YYYY-MM-DD-short-title.mdx      # One entry = one file
└── ...
src/components/ChangelogIndex.astro # Generates the cards from the collection
```

- **One entry = one file**, named `YYYY-MM-DD-kebab-title.mdx`. The date prefix is mandatory: it is the index's sort key.
- **The index is auto-generated** from the content collection. Adding an entry = creating the file. Nobody ever edits `index.mdx` → zero double writing, zero conflicts.

## The index component (Astro Starlight)

```astro
---
// src/components/ChangelogIndex.astro
// Sorting by id IS the reverse-chronological sort, thanks to the YYYY-MM-DD
// prefix in the file name — no date parsing.
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

And `index.mdx` is reduced to frontmatter + an intro paragraph + `<ChangelogIndex />`.

> In other generators (fumadocs, Docusaurus, VitePress) the mechanism changes but the pattern is the same: the index is derived from the files, never maintained by hand.

## Entry format

```mdx
---
title: "Month DD, YYYY - Short Title"
description: A 1-2 line summary — this is the index card's text, keep it short
date: YYYY-MM-DD
tags:
  - changelog
  - relevant-tags
---

Introduction paragraph.

---

## type: Change Title

### Changes
- What changed, concretely

### Files Changed
paths/touched

### Decision Rationale
Why, trade-offs, rejected alternatives.
```

Hard rules:

- **A 1-2 line `description`.** The narrative lives in the entry body; the index card only announces. If the description needs a paragraph, the title is wrong or the entry covers too much.
- **No commit-hash tables.** They do not exist until the merge, and release-please already covers that layer.
- **`Decision Rationale` is the mandatory section** — it is the only information that does not live anywhere else (not in the git log, not in the diff, not in release-please).

## The bar: what deserves an entry

Entry **yes** (there is a decision or a lesson to record):

- New features, architecture changes, breaking changes
- Schema changes — especially migrations and their history in prod
- Fixes with a non-obvious root cause, or with a lesson
- **Postmortems** — something went wrong in deploy/prod and how it was recovered (the most valuable entries of the pattern)

Entry **no** (release-please already records it):

- Trivial fixes, typos, dependency bumps, mechanical refactors with no story

## Relationship with release-please

Two complementary layers, no overlap:

| Layer                    | Who writes it                             | What it tells                        |
| ------------------------ | ----------------------------------------- | ------------------------------------ |
| `CHANGELOG.md` per app   | release-please (automatic)                | Which commits went into which version |
| `docs/changelog/`        | the author or the agent (manual, with a bar) | Why, what was rejected, lessons    |

If a change has no "why" to tell, it does not duplicate the layer: it lives only in the release-please one.
