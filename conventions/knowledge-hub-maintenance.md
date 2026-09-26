# Knowledge hub maintenance

**The hub is linked, not copied — so every change to it is a change to every repo that links
to it.** Product repositories point at hub pages by URL instead of carrying their own copy of
reusable knowledge. That is what keeps one source of truth, and it is also what makes a careless
edit expensive: a renamed page, a split section or a product detail promoted into the hub
reaches every consumer at once, silently.

The executable version of this convention is the
[`cs-update-knowledge-hub` skill](https://github.com/niway-dev/skills/tree/main/skills/cs-update-knowledge-hub).
This page is the reasoning; the skill is the procedure. Deciding *whether* a lesson belongs in
the hub at all is [lesson-distillation.md](./lesson-distillation.md); this page covers how it
lands once it does.

## The hub has consumers you cannot see from inside it

Relative links inside the hub are easy to check. The links that break are the **absolute URLs in
other repositories** — ADRs, specs, READMEs, `CLAUDE.md` files, workflow comments and agent
skills, all of the form `github.com/csdev19/general-knowledge/blob/main/<path>`. A search inside
the hub finds none of them. Measured across five sibling repositories: 37 to 72 hub URLs each,
and a single page in `monorepos/` was linked from twelve files outside the hub, one of them a
workflow.

So moving or renaming a page is a cross-repository change. The default is to **keep the old path
as a one-paragraph stub** pointing at the new one, and update the consumers you can reach in the
same sitting, one PR per repository. A stub costs one small file; a broken link in an ADR costs
the next reader the reasoning the ADR was written to preserve.

## Generalise, then keep the evidence honest

A page states the pattern with placeholders (`<app>`, `<scope>`, `apps/desktop`), because the
reader is building a different product. The product a lesson came from can appear only where the
evidence is — a section explicitly labelled as a measured example, with its numbers — never in
the rule itself. A rule that names one product reads as that product's configuration.

That exception covers only the owner's own products. The hub is public, and much of what reaches
it is learned at an employer's or a client's: their name, products, repositories, people, ticket
systems and internal tooling never appear in it, not even as evidence. They become placeholders,
a real name survives only when the owner explicitly asks for it, and the owner sees the final
text before it is pushed. [lesson-distillation.md](./lesson-distillation.md) carries the rule.

## Extend before adding

Most new knowledge is a missing section of a page that exists, or a slot a folder index already
lists as "not written yet". A new page earns its place when it has its own reader and its own
reason to be linked. `stacks/` recipes compose topic pages into a reading list plus assembly
notes; they never own content a topic page should.

## An unindexed page does not exist

Readers enter through indexes: each folder's `README.md`, the root README's topic and stack
tables, and this folder's agent-skills list for pages that pair with a skill. A page missing from
its index is found only by search, which is how duplicates get written.

## Language

New pages and rewritten sections are written in English. Existing Spanish pages are translated
when they are substantially rewritten, not in passing — a drive-by translation hides the real
change in a large diff.

## Pairing with the skills repo

When a page is the reasoning of a skill, a change to a rule changes both, in the same sitting,
and each PR names the other. A change to how a step is done touches only the skill.
