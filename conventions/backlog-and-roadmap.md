# Backlog and roadmap: issues as the backlog, the roadmap as a promise

> **One-minute summary.** The backlog is the GitHub issue list, not a folder: an idea is
> captured as an issue labelled `backlog` the moment it appears, and the label plus
> `author:@me` *is* the index — nothing to keep in sync by hand. The roadmap is a separate,
> much smaller file per product: only the promise (phases and one line per promised feature,
> linking its issue), edited when promising and never at ship. Status is always **derived**
> from the issue and its PRs, never written down anywhere — that removes the one file every
> shipping PR would otherwise fight over.

The executable version of the capture half of this convention is the `cs-capture` skill
(**planned** — not built yet). This page is the reasoning; the skill is the procedure.

## Backlog: capture as a GitHub issue

Capture runs `gh issue create` and touches nothing else on disk — no branch, no worktree, the
current work stays untouched. The issue body is fixed:

```markdown
## What          the feature or rework, in the requester's own words — a story, not "fix a button"
## Why           the problem; what happens if this is never done
## What I know    every detail already available: files, screens, constraints, links, pasted text
## Open           what is not known yet
## Related        ADRs, feature docs, hub pages, sibling issues in other repositories
```

- **Label:** `backlog`. **Title:** the feature name.
- **Index:** `gh issue list --label backlog --author @me` — a derived view, never a hand-kept
  file, so it cannot go stale.
- **Cross-repo ideas** get one issue per repository they change, cross-linked in each one's
  Related section.
- **Not a capture:** a one-line fix doable right now, or work that belongs to the PR already in
  progress.
- **The item grows into the spec.** When design starts, it reads the issue body as its input;
  the spec's status line links back to the issue; the design PR references it; the shipping PR
  closes it. Questions asked before design starts are answered on the issue thread.

### Public-repo rule

On a public repository, anyone can open an issue, but only collaborators can apply a label.
`label:backlog author:@me` is what isolates the owner's own items from outsiders' issues — and
that isolation only holds if the label is applied by a person, not by automation open to
everyone. **Never ship an issue template whose front matter applies the `backlog` label**: a
template applies its labels to whoever files through it, which would let any external
contributor's issue into the index the owner reads as their own backlog.

## Roadmap: the promise, with derived status

One `docs/roadmap.md` per product:

```markdown
# <Product> roadmap

> What this product has committed to build next, by phase. Status is derived from each issue
> and its linked PRs — nothing here says "done". What exists is in the feature docs index; why
> it is that way is in the ADR log.

## Phase 1 — <name>
- <Feature A> — #12
- <Feature B> — #15

## Phase 2 — <name>
- <Feature C> — #18 (after <dependency>)
```

- **Edited only when promising** — when picking an item up with "let's do it" or similar, or
  when a captured item is promoted onto a phase. In that same edit, prune lines whose issues
  have already closed.
- **The shipping PR never touches this file.** That is the whole point: a file every shipping
  PR edits is a merge-conflict magnet (see [changelog-pattern.md](./changelog-pattern.md) for
  the same lesson learned once already, on an index). Keeping the roadmap promise-only means
  two features shipping at once cannot collide on it.
- Never deleted as a file; shipped and dropped lines are the only things pruned from it, and
  only at the next promote.

### Status derivation

Nothing writes a status field. It is read off the issue and its PRs, as a query run whenever
someone asks "where are we":

| Signal | Status |
| --- | --- |
| Issue open, no PR references it | promised |
| An open or merged `design: …` PR references it, no code PR yet | designing |
| An open PR that closes it | building |
| Issue closed as completed | shipped |
| Issue closed as not planned | dropped — the reason is the close comment |

This is the same pattern for every repository that adopts it: one roadmap file, one issue
list, and a query instead of a maintained field.

## Related work that is not this pattern

[plan-to-backlog.md](./plan-to-backlog.md) converts an *approved plan* into a set of
self-sufficient runner documents for parallel execution — a different `backlog/` folder, for a
different purpose (splitting one plan into disjoint parallel work), not the personal
item-capture backlog this page describes. Do not conflate the two: an issue captured here
becomes a spec and a plan; that plan, if it needs to fan out to several agents at once, is what
`plan-to-backlog.md` turns into runner documents.

## Related

- [design-workflow.md](./design-workflow.md) — what a backlog item becomes once picked up.
- [changelog-pattern.md](./changelog-pattern.md) — the earlier lesson (a hand-kept index is a
  merge-conflict magnet) that the promise-only, derived-status roadmap applies again here.
- [plan-to-backlog.md](./plan-to-backlog.md) — a different, later-stage use of the word
  "backlog"; see the note above.
- [cross-machine-sessions.md](./cross-machine-sessions.md) — an idea for other work, noticed
  mid-session, is a capture, not a park.
