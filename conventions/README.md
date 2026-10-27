# Conventions

Reusable, product-agnostic conventions and workflow patterns distilled from real project docs.

## Architecture patterns

- **[constants-pattern.md](./constants-pattern.md)** — Type-safe enum-like constants: const object + derived type + values array + type guard.
- **[enums-as-const.md](./enums-as-const.md)** — Never use TS `enum`; use `as const` arrays/objects with types derived via `typeof`.
- **[schemas-first.md](./schemas-first.md)** — Define Zod (v4) schemas in the domain package and reuse them across backend and frontend.
- **[tagging-system.md](./tagging-system.md)** — Two-tier tags: default tags in code (slug/label/emoji/color) + per-scope custom tags with a curated color palette. One source of truth across web/mobile/widget.

- **[app-naming.md](./app-naming.md)** — One name per app, `<product>-<role>` (`<product>-api`, `<product>-web`, `<product>-desktop`), used for its directory, package, deployed resource and secrets tag; detail goes after it (`<product>-desktop-sign`); never the environment or the framework.
- **[app-bundle-identifiers.md](./app-bundle-identifiers.md)** — Reverse-DNS bundle IDs under `dev.niway.*` (niway.dev), frozen after first release; why users never see them and why shipped exceptions (Kaipu's `com.niway.*`) are grandfathered rather than migrated; renaming an Electron app without moving its `userData` folder.

## Data & delivery

- **[data-sourcing-and-seeding.md](./data-sourcing-and-seeding.md)** — Filling an app with real, structured, geo-tagged reference data (e.g. attractions): why an agent beats a plain chat, legal sources (Wikidata CC0 primary, never scrape TripAdvisor), the curate→resolve→enrich→normalize→validate→seed pipeline, and provenance as a field.
- **[tool-doctor-pattern.md](./tool-doctor-pattern.md)** — Global tools `bun install` cannot provide (secrets CLI, cloud CLI): one script that **checks and reports, never installs**. Why installing is a supply-chain surface with no checksum to verify against, the presence → version → **access** ladder, and why one check that proves the chain beats two that can disagree.
- **[ci-cd-pipeline-strategy.md](./ci-cd-pipeline-strategy.md)** — Trunk + tags, one `verify` gate run by **lefthook pre-push** and by the PR as backstop, tiered checks; the native/mobile install is the biggest CI cost (split it out); `deps:weight` protocol. **Normative rules now in [monorepos/release-gated-verification.md](../monorepos/release-gated-verification.md)**; runner pricing in [monorepos/ci-runner-cost.md](../monorepos/ci-runner-cost.md).

## Workflow conventions

- **[mvp-first-then-refactor.md](./mvp-first-then-refactor.md)** — Two-phase feature workflow: ship an inline MVP end-to-end first, then extract the domain/application/infra layers once it's stable. When to refactor.
- **[changelog-pattern.md](./changelog-pattern.md)** — Docs-app changelog as a decision journal: one file per entry, auto-generated index (no manual index = no merge conflicts), entry bar, and how it complements release-please. _(Spanish)_
- **[plan-to-backlog.md](./plan-to-backlog.md)** — Converting an approved plan into self-sufficient backlog deliverables that runner agents execute in parallel; the plan becomes superseded. _(Spanish)_
- **[feature-docs-playbook.md](./feature-docs-playbook.md)** — Adopting feature docs: pick three features, one flat file each, a generated index, the update rule in the repo's instructions; guardrails (state not history, 100 lines, links not copies, gotchas expire); how a trial is judged, keep or delete. Experiment started 2026-10-19.
- **[working-with-the-owner.md](./working-with-the-owner.md)** — How the hub's owner thinks, decides, and wants to be worked with: context dumped in one pass and expected processed, prose options with a counterargument, one free-text reply, coarse-grained approval, decisions written down with a reopen condition.

## Agent skills

Each convention below has an executable counterpart in the
[skills repo](https://github.com/niway-dev/skills): the page here is the
reasoning, the skill is the procedure, and they link both ways. When a skill
changes because something worked or failed in practice, its page here changes
with it.

- **[development-pipeline.md](./development-pipeline.md)** — From approved spec to merge-ready PR with no questions mid-run: a router for live sessions and playbooks for the queue, one gate (proof row per criterion, verdict tied to the head, low risk merges alone), risk classified by the verifier, a review loop budgeted in rounds, per-platform verification with screenshots and video. Skill: `cs-autopilot` (**planned**).
- **[design-workflow.md](./design-workflow.md)** — Capture → roadmap promise → design (research always, an ADR-contradiction check, options in prose, a hard gate) → spec + 0..n ADRs in their own PR → an ephemeral plan deleted at ship. Skill: `cs-design` (**planned**).
- **[adr-pattern.md](./adr-pattern.md)** — Format (context, decision, alternatives rejected, consequences, non-goals, reopen condition), when a decision needs one, how it links to the spec and the feature doc, and why there is no hand-kept ADR index (Status lines are the source; any list is generated). Paired reasoning for whichever skill writes ADRs (`cs-design`, **planned**).
- **[backlog-and-roadmap.md](./backlog-and-roadmap.md)** — The backlog is the GitHub issue list (label `backlog`, `gh issue list --label backlog --author @me` as the index); the roadmap per product is promise-only, edited when promising and never at ship; status is always derived. Skill: `cs-capture` (**planned**).
- **[feature-docs-as-memory.md](./feature-docs-as-memory.md)** — One flat file per feature even across layers, generated index, the "Where it lives" paths table as a contract the ship step checks, `lastVerified` stamp. Skill: `cs-document-feature` (**planned**; replaces the template `feature-docs` skill and `save-feature` command).
- **[repo-briefings.md](./repo-briefings.md)** — A repo explains itself to each kind of reader from its own truth: audience-split briefings, a per-repo binding file, date stamps as promises, audit mode that writes nothing. Skill: `cs-generate-briefings`.
- **[verifiable-handoffs.md](./verifiable-handoffs.md)** — Finished work handed back with the means to check it. Skill: `cs-write-handoff` (gaining a ship step — feature doc update, plan deletion, closing the backlog issue — **planned**).
- **[evidence-driven-optimization.md](./evidence-driven-optimization.md)** — Three artifacts in order: an evidence file (provenance, workload, runtime, code version, what each check verified), a budget table (measured, projected and bounded rows kept apart, matching metrics, tested envelope, margin as policy, four verdicts including undetermined), an optimization record (explicit objective, attribution experiments as evidence not accounting, prioritization by contribution to the objective, complementary correctness checks). Skills: `cs-capture-evidence`, `cs-check-budget`, `cs-optimize-with-evidence`.
- **[audit-briefs.md](./audit-briefs.md)** — Work briefed for a fresh-context agent to attack: intent, claims sorted by how they are known, decisions with owners, and the writer's own weakest points. Skill: `cs-write-audit-brief`.
- **[cross-machine-sessions.md](./cross-machine-sessions.md)** — Parking a session on one machine and picking it up on another through the repo: a fixed restart file, one disposable checkpoint commit, explicit restore. Skills: `cs-park`, `cs-pickup`.
- **[lesson-distillation.md](./lesson-distillation.md)** — What a session taught goes to exactly one home: the skill, the hub page, a new skill, or the project. As a conditional, not a story; tested on a fresh agent before shipping. Skill: `cs-distill-lesson`.
- **[review-feedback.md](./review-feedback.md)** — A review comment is a claim to verify: six kinds, one treatment each; settled decisions are escalated, not reopened; replies cite commits. Skill: `cs-address-review-feedback`.
- **[test-audits.md](./test-audits.md)** — Trimming tautological and change-detector tests and adding the ones that catch the failures the owner fears; the failure list comes before the test. Skill: `cs-audit-tests`.
$1Consult the hub before deciding and correct a page where a failed hypothesis lives. Skill: `cs-knowledge-hub`.
- **[writing-with-ai.md](./writing-with-ai.md)** — Publishing technical writing drafted with AI: the author's words quoted verbatim from a raw input file and marked on the page, the rest drafted and disclosed, a lint that fails a rewritten author block and the known style tells. Skills: `cs-write-blog-post`, `cs-audit-blog-post`.

## Agent delegation

- **[ai-agent-delegation.md](./ai-agent-delegation.md)** — Two delegation modes (fast/expensive vs slow/cheap), which roles never get cheaper, how to dispatch without blowing up the orchestrator's context, and measured evidence that process structure — not model tier — is what makes delegated work slow. Includes the "name the unit before quoting the number" measurement lesson. _(Spanish)_
- **[ai-model-usage.md](./ai-model-usage.md)** — Model roles (Opus 5.5 default and orchestrator, Fable as refiner that never explores, Sonnet executor and reviewer), when parallel agents pay and concurrency defaults per model, the Opus-draft → Fable-refine → Sonnet-implement pipeline, where agent tokens really go (context re-read per turn, cache expiry on resume), protocols for a fan-out on a capped model and for recovering from a usage limit, the tool kit a reviewer agent needs, an experiment to measure Sonnet vs Fable reviews, and a measured remediation session.
