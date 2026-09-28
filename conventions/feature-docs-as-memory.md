# Feature docs as the agent's memory

> **One-minute summary.** One flat file per feature — even when it spans several layers or
> modules — is what an agent reads instead of rereading the code. Its index is generated, never
> hand-edited, so two shipping PRs can never collide on it. A fixed "Where it lives" paths
> table is the contract: the ship step greps the diff against those paths, and a path that no
> longer exists fails the check. The doc is updated by the ship step of whatever PR touches it,
> and carries a `lastVerified` stamp so a stale doc is visible, not silently trusted.

The executable version of this convention is the `cs-document-feature` skill (**planned** —
replaces a template's `feature-docs` skill and `save-feature` command once built). This page is
the reasoning; the skill is the procedure.

## Why one flat file, cross-module

A feature is rarely one layer. Splitting its documentation by domain / application / infra /
UI scatters the one thing an agent actually needs — "how does this feature work, end to end" —
across files that must each be found and reconciled. One file per feature, regardless of how
many layers it touches, is what lets an agent "talk to the code without rereading all of it
every time": read the index, read the one feature doc, read only the paths it names.

## Shape

```mdx
---
title: <Feature>
description: <one line — this is the index card>
lastVerified: YYYY-MM-DD        # set by the ship step; the index sorts and flags stale docs
tags: [<module>, <module>]      # area is a tag, not a folder
---

## What it does
2-4 sentences, from the user's perspective.

## Where it lives
| Layer | Path | Entry point |
| --- | --- | --- |
domain · application · infra · UI · tests — this table is the contract (see below).

## How it works
The flow, roughly ten lines; a diagram only when it branches.

## Why it is this way
| Decision | ADR / spec |
| --- | --- |
Links only — the reasoning lives at the link, never restated here. See
[adr-pattern.md](./adr-pattern.md).

## Gotchas
What bit an agent or a user, each with the PR that fixed it.

## History
| Date | PR | What changed |
| --- | --- | --- |
Newest first, one line each.
```

## The paths table is the contract

"Where it lives" is not documentation of the code — it is a check on it. The ship step greps
the shipping PR's diff against every path listed there; a path that no longer exists in the
repo fails the check, which is what catches a feature doc silently going stale the moment its
code moves. An agent starting a task reads the feature index, then the one feature doc that
matches, then only the paths that doc's table names — never the whole codebase looking for
context that already has a home.

## The generated index

The index is generated from the docs collection — sorted, for instance, by `lastVerified` or
by title — the same way a docs-app changelog's index is generated from its entries rather than
hand-maintained (see [changelog-pattern.md](./changelog-pattern.md), which records the exact
failure this avoids: an index every writer edits at the same spot is a merge-conflict magnet,
and a hand-copied card next to the real entry is a second copy that drifts). Adding a feature
doc is creating the file; nobody edits an index by hand, so two features shipping at once never
collide on one.

Two shipping PRs editing the **same** feature's doc is a normal, expected conflict and is
accepted — that is different from two PRs colliding on a shared index file that neither of them
is actually about.

## When it is updated

The ship step — a planned addition to the handoff convention, see
[verifiable-handoffs.md](./verifiable-handoffs.md) — checks the diff against every doc's paths
table: a touched path updates that doc's relevant section and its `lastVerified` stamp; a new
feature with no doc yet gets one created. A feature doc is never updated "later" — the moment a
diff touches its paths is the only moment it is meant to change.

## Related

- [design-workflow.md](./design-workflow.md) — the spec and ADRs a feature doc's "Why it is
  this way" table links to, rather than restates.
- [adr-pattern.md](./adr-pattern.md) — the format behind those links.
- [changelog-pattern.md](./changelog-pattern.md) — the generated-index lesson this convention
  reapplies.
- [verifiable-handoffs.md](./verifiable-handoffs.md) — the ship step that updates a feature doc.
