# Conventions

Reusable, product-agnostic conventions and workflow patterns distilled from real project docs.

## Architecture patterns

- **[constants-pattern.md](./constants-pattern.md)** — Type-safe enum-like constants: const object + derived type + values array + type guard.
- **[enums-as-const.md](./enums-as-const.md)** — Never use TS `enum`; use `as const` arrays/objects with types derived via `typeof`.
- **[schemas-first.md](./schemas-first.md)** — Define Zod (v4) schemas in the domain package and reuse them across backend and frontend.
- **[tagging-system.md](./tagging-system.md)** — Two-tier tags: default tags in code (slug/label/emoji/color) + per-scope custom tags with a curated color palette. One source of truth across web/mobile/widget.

- **[app-bundle-identifiers.md](./app-bundle-identifiers.md)** — Reverse-DNS bundle IDs under `dev.niway.*` (niway.dev), frozen after first release; why users never see them and why shipped exceptions (Kaipu's `com.niway.*`) are grandfathered rather than migrated.

## Data & delivery

- **[data-sourcing-and-seeding.md](./data-sourcing-and-seeding.md)** — Filling an app with real, structured, geo-tagged reference data (e.g. attractions): why an agent beats a plain chat, legal sources (Wikidata CC0 primary, never scrape TripAdvisor), the curate→resolve→enrich→normalize→validate→seed pipeline, and provenance as a field.
- **[tool-doctor-pattern.md](./tool-doctor-pattern.md)** — Global tools `bun install` cannot provide (secrets CLI, cloud CLI): one script that **checks and reports, never installs**. Why installing is a supply-chain surface with no checksum to verify against, the presence → version → **access** ladder, and why one check that proves the chain beats two that can disagree.
- **[ci-cd-pipeline-strategy.md](./ci-cd-pipeline-strategy.md)** — Trunk + tags, one `verify` gate run by **lefthook pre-push** and by the PR as backstop, tiered checks; the native/mobile install is the biggest CI cost (split it out); `deps:weight` protocol. **Normative rules now in [monorepos/release-gated-verification.md](../monorepos/release-gated-verification.md)**; runner pricing in [monorepos/ci-runner-cost.md](../monorepos/ci-runner-cost.md).

## Workflow conventions

- **[mvp-first-then-refactor.md](./mvp-first-then-refactor.md)** — Two-phase feature workflow: ship an inline MVP end-to-end first, then extract the domain/application/infra layers once it's stable. When to refactor.
- **[backlog-pattern.md](./backlog-pattern.md)** — How a `backlog/` folder maps deferred work: status legend, index table, one item per file. _(Spanish)_
- **[changelog-pattern.md](./changelog-pattern.md)** — Docs-app changelog as a decision journal: one file per entry, auto-generated index (no manual index = no merge conflicts), entry bar, and how it complements release-please. _(Spanish)_
- **[specs-and-plans-workflow.md](./specs-and-plans-workflow.md)** — The brainstorm → spec → plan → archive flow, naming convention, and how it pairs with the backlog. _(Spanish)_
- **[plan-to-backlog.md](./plan-to-backlog.md)** — Converting an approved plan into self-sufficient backlog deliverables that runner agents execute in parallel; the plan becomes superseded. _(Spanish)_

## Agent skills

Each convention below has an executable counterpart in the
[skills repo](https://github.com/niway-dev/skills): the page here is the
reasoning, the skill is the procedure, and they link both ways. When a skill
changes because something worked or failed in practice, its page here changes
with it.

- **[repo-briefings.md](./repo-briefings.md)** — A repo explains itself to each kind of reader from its own truth: audience-split briefings, a per-repo binding file, date stamps as promises, audit mode that writes nothing. Skill: `generate-briefings`.
- **[verifiable-handoffs.md](./verifiable-handoffs.md)** — Finished work handed back with the means to check it. Skill: `write-handoff`.
- **[audit-briefs.md](./audit-briefs.md)** — Work briefed for a fresh-context agent to attack: intent, claims sorted by how they are known, decisions with owners, and the writer's own weakest points. Skill: `write-audit-brief`.
- **[cross-machine-sessions.md](./cross-machine-sessions.md)** — Parking a session on one machine and picking it up on another through the repo: a fixed restart file, one disposable checkpoint commit, explicit restore. Skills: `cs-park`, `cs-pickup`.
- **[lesson-distillation.md](./lesson-distillation.md)** — What a session taught goes to exactly one home: the skill, the hub page, a new skill, or the project. As a conditional, not a story; tested on a fresh agent before shipping. Skill: `distill-lesson`.
- **[review-feedback.md](./review-feedback.md)** — A review comment is a claim to verify: six kinds, one treatment each; settled decisions are escalated, not reopened; replies cite commits. Skill: `address-review-feedback`.
- **[test-audits.md](./test-audits.md)** — Trimming tautological and change-detector tests and adding the ones that catch the failures the owner fears; the failure list comes before the test. Skill: `audit-tests`.

## Agent delegation

- **[ai-agent-delegation.md](./ai-agent-delegation.md)** — Two delegation modes (fast/expensive vs slow/cheap), which roles never get cheaper, how to dispatch without blowing up the orchestrator's context, and measured evidence that process structure — not model tier — is what makes delegated work slow. Includes the "name the unit before quoting the number" measurement lesson. _(Spanish)_
