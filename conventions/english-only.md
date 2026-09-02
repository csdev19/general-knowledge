# English only

**Everything written into a repository is in English.** This hub, every project that links to
it, and every agent working on either one.

## Scope — no exceptions

| Surface                                                              | Language    |
| -------------------------------------------------------------------- | ----------- |
| Docs, specs, ADRs, READMEs, changelog entries, backlog items          | **English** |
| Code: identifiers, comments, log messages, test names, error messages | **English** |
| Commit messages, branch names, PR titles, PR bodies, review comments  | **English** |
| Issue titles and bodies, doc-site copy                                | **English** |
| Agent skills, prompts, and the tables/legends they reference          | **English** |

The **one exception** is *localized product copy* — the strings a user reads in the product,
living in an i18n message catalog (`messages/es.json`, `es-PE.json`, …) or in a per-locale email
template. Those are data. Everything around them — the keys, the file structure, the docs
explaining them, and the comments next to them — stays English.

## Why

- **Reuse.** This hub is the single source of truth across every project. A doc written in
  Spanish cannot be pasted into an English codebase, quoted in an English PR, or handed to a
  collaborator who does not read Spanish.
- **Mixed files are the worst outcome.** Spanish prose around English identifiers, or an English
  table with `🔵 Propuesto` in its cells, forces every reader to context-switch mid-line and makes
  grep unreliable (two words for the same concept).
- **Agents copy what they see.** An agent reading a Spanish exemplar reproduces Spanish. One
  Spanish doc seeds Spanish across a whole repo, one generation at a time.
- **Terms of art are already English.** issuer, claims, refresh, upsert, hot path, trusted origin.
  Writing Spanish prose around them produces neither good Spanish nor good English.

## The rule that is easy to get wrong

**The language of the conversation is not the language of the artifact.** Talking with the user in
Spanish is fine and expected — reply in whatever language they use. What gets *written into the
repo* is still English, always. The same applies to a Spanish source document (a vault note, a
voice memo, a chat thread): it is translated on the way in, not pasted.

## Applying it to an existing repo

1. Sweep for Spanish prose in `*.md` / `*.mdx` and in code comments.
2. Translate whole files rather than sentence by sentence — half-translated docs read worse than
   either language alone.
3. Translate the **tokens too**, not only the prose: status legends (`🔵 Propuesto` → `🔵 Proposed`),
   table headers (`Ítem | Área | Estado` → `Item | Area | Status`), effort scales
   (`Bajo/Medio/Alto` → `Low/Medium/High`). Any skill or template that writes those tokens has to
   be updated in the same pass, or it will re-seed the Spanish version.
4. State the rule in the repo's `CLAUDE.md` so future sessions inherit it, and link back here.
