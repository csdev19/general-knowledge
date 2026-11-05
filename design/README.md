# Design: the owner's taste profile

_How the owner wants every UI an agent builds on his personal projects to look, kept as principles, a never list, a reference bank and one page per app type._

**This folder is owner-specific.** Unlike most of the hub, it records one person's taste, in the same
way [working-with-the-owner.md](../conventions/working-with-the-owner.md) records one person's working
style. It applies to the owner's **personal projects only**, never to an employer's or a client's
repository, and it names no company product. Someone else can run the same skill against their own
profile.

## Three layers

| Layer | Holds | Lives in |
|---|---|---|
| Procedure | gate, load, plan, build, words pass, critique, learn | the [`cs-apply-taste` skill](https://github.com/niway-dev/cs-tack/blob/main/skills/cs-apply-taste/SKILL.md) in the skills repo |
| Profile | principles, never list, references, one page per app type | this folder |
| Application | this product's register, tokens or their source, overrides | the project's `design-brief` |

The skill holds no taste of its own. Decisions:
[ADR 0019](https://github.com/niway-dev/cs-tack/blob/main/docs/adr/0019-taste-precedence-brief-profile-platform-then-frontend-design.md)
(precedence) and
[ADR 0020](https://github.com/niway-dev/cs-tack/blob/main/docs/adr/0020-taste-profile-in-the-public-hub-grown-by-confirmed-write-back.md)
(where the profile lives and how it grows), from the
[apply-taste spec](https://github.com/niway-dev/cs-tack/blob/main/docs/specs/2026-11-03-apply-taste-design.md).

## Precedence

For product UI: project `design-brief` > this profile > `frontend-design` defaults. Two things sit
outside that chain: **platform conventions** (Apple HIG, Material, the terminal's conventions) come
first on mobile, desktop and CLI, and taste works inside them; **accessibility** is a floor nothing
above it overrides.

## Two registers

- **Calm** is the default: restrained, "simple, with character". Character comes from one deliberate
  element inside the calm register, for every product including landings.
- **Playful / imperfect** applies only to editors, sandboxes and things made to tinker with. See
  [playful-tools.md](./playful-tools.md). The app type sets the default; the brief names the register
  for the product.

## Pages

| Page | Holds |
|---|---|
| [principles.md](./principles.md) | calm by default, the words rule, tokens source of truth |
| [never.md](./never.md) | what the owner does not want, each with a one-line why |
| [references.md](./references.md) | the reference bank: link, register, app types, the owner's note |
| [landing.md](./landing.md), [dashboard.md](./dashboard.md), [cli.md](./cli.md), [mobile.md](./mobile.md), [desktop.md](./desktop.md), [playful-tools.md](./playful-tools.md) | entries per app type |

## Entry template

```markdown
### Badges or a segmented control instead of a select
- **When:** few visible options and frequent switching.
- **When not:** more than about seven options, or a dynamic list → combobox.
- **Reference:** [Linear filters](https://linear.app) — the options are visible, switching is one click.
- **Since:** 2026-11-03. Refined: —
```

An entry is a preference written as an instruction, with its boundary. Adjectives without a boundary
("clean", "modern") are not entries. A rejection becomes a `never.md` entry or a "when not" on the
entry it contradicts. A refinement edits the entry in place and sets `Refined:`; git keeps the history.

## Write-back rule

Entries change **only after the owner confirms**. A reaction in a session produces one proposed entry
and one prompt (add as written, or refine first); nothing is written before the owner answers. A
confirmed entry goes through the `cs-knowledge-hub` skill: consult first, then add or correct in place.

## References are links and notes

References are links plus the owner's one-line notes. No screenshot, download or copy of a referenced
product or of Awwwards or Mobbin content enters this hub; the images belong to their owners.
