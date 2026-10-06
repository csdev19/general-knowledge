# Linear workflow: the tracker holds what, why and status; the repo holds how

> **One-minute summary.** One Linear workspace is the single tracker for every repository
> the owner works on, personal or paid. Linear owns the mutable part of a piece of work:
> the idea, the why, priority, status, progress notes. The repository owns the durable
> part: architecture, ADRs, specs, contracts, code. GitHub is the history. The Linear issue
> ID is the one identifier that ties branch, spec, commit and PR together. Capture is
> deliberately light (a title and two lines, from any assistant, including a chat bot);
> grooming is where structure is added. Teams are split by **who pays**, projects by product
> or client, and a third mode of work never gets a team: it gets a project label.

The executable version of this convention is the `cs-linear` skill (**planned**). This page
is the reasoning; the skill is the procedure. It supersedes the capture half of
[backlog-and-roadmap.md](./backlog-and-roadmap.md) for repositories that adopt it; the
GitHub-issues path stays for repositories where outsiders file issues.

## The boundary

| Linear owns | The repository owns |
| --- | --- |
| ideas, feature requests, bugs | code |
| roadmap, priorities, milestones | architecture, ADRs, technical design |
| status, progress notes, blockers | specs, API contracts, data models |
| product-level acceptance criteria | technical investigation, testing strategy |
| why something should exist | durable engineering knowledge |

Rule of thumb: **temporal or mutable → Linear; durable technical knowledge → Git.** A Linear
issue carries enough to understand the problem and the desired outcome. Engineering detail
is linked, never pasted: a long technical document copied into Linear is a second copy that
will drift.

## The identifier

The Linear ID (`<KEY>-<n>`) is the common key across the three systems:

```text
Linear issue      KEY-143
Branch            feat/KEY-143-live-annotations
Technical spec    docs/features/KEY-143-live-annotations.md
Commit            feat(KEY-143): add annotation canvas
Pull request      KEY-143: Live annotations
```

Linear's GitHub integration reads the ID from branch names, PR titles and commit messages and
moves the issue (branch → In Progress, PR → In Review, merge → Done) without anyone editing
status by hand. Existing branch and commit conventions are kept; the ID is added to them,
not replaced.

## Workspace structure (Free plan: two teams, 250 non-archived issues)

**Team = who pays and who owns the workflow.** A team is a workflow boundary, with its own
states, labels, triage inbox and ID prefix. Two are enough and the Free plan allows no more:

| Team | What goes in it |
| --- | --- |
| Personal | the owner's own products and tooling |
| Work | every paid relationship: freelance clients and company employment |

**Project = one product, client or employer.** Projects are cheap: a name and a filter.
"personal → `<product>`" is one click; "everything paid" is the other team.

**A third mode of work gets a project label, not a team.** Project label group `mode`:
`personal`, `freelance`, `company`. Filtering projects by `mode` answers "all company work"
without spending a team.

**Company projects hold only the owner's map.** An employer has its own tracker, and that
tracker is the source of truth for *their* tickets. The owner's project for them holds one
issue per thing they are on (linking the employer's ticket), progress notes as comments, and
ideas about their product. Never their code, credentials or internal detail.

**Issue labels, workspace-level** so both teams share them:

| Group | Labels | Meaning |
| --- | --- | --- |
| type | `Idea`, `Feature`, `Bug`, `Improvement`, `Chore` | `Idea` is raw and ungroomed; grooming turns it into another type or closes it |
| `source` | `hermes`, `claude`, … | which assistant captured the issue |

No Linear templates: the shape of each type lives in the skill, so a capture with almost no
information is still valid.

## Capture, then groom

**Capture is a dump.** From any assistant, including a chat-bot one on a phone: a title,
two or three lines, the project if known, label `Idea`, the `source` label. It lands in the
team's **Triage** inbox, not in the backlog. No acceptance criteria, no implementation detail.

**Grooming empties Triage.** Each item becomes a typed issue with a project, priority and
enough context (problem, expected outcome, acceptance criteria, link to the repo or spec), is
merged into an existing one, or is closed with a reason. Grooming is also when the monthly
archive runs (see limits).

## Working a ticket from a repository

`work on KEY-143` means: read the issue (goal, acceptance criteria), inspect the repository
and its docs, find or create the technical spec when the work has real design decisions or
spans several areas (`docs/features/<ID>-<slug>.md`; small fixes work straight from the
issue), implement, validate, and write back to Linear only what is useful: a short
implementation summary and links. `continue KEY-143` rebuilds context from the Linear issue,
the spec, the current branch and the git history, never from a previous conversation: the
workflow must be recoverable from persistent artifacts.

## Limits to manage on the Free plan

- Closed issues count toward the 250 until they are archived; only archived ones do not.
- Auto-archive (per team, 1 to 12 months) does not fire while the issue's project is not
  completed, and product projects never complete. So a **monthly manual archive** of Done and
  Canceled issues is part of grooming, and auto-archive is set to one month as a backstop.
- When the cap bites, the paid tier for a single user costs less than one afternoon of
  building anything; an own tracker is an idea to revisit after months of real use, not a
  plan.

## Assistants

Every assistant reaches the same workspace through Linear's hosted MCP server
(`https://mcp.linear.app/mcp`, OAuth). The capture-only assistant (the chat bot) gets an
explicit tool allowlist: read everything, create or edit issues and comments; no deletes, no
project or label administration. Those stay with the coding assistant and the Linear UI.

**OAuth trap with more than one Linear account.** The login opens the default browser, and
whatever Linear session is active there is approved instantly, without a workspace picker.
On a new machine, sign in to Linear as the account that owns the workspace *before* running
the assistant's login, then verify with a read-only call that returns the workspace URL.

## Measured example

The owner's workspace, 2026-10-05: team `Niway` (personal; projects Kaipu, dotfiles, skills)
and team `Work` (paid; one project per client or employer). Hermes (Telegram) captures with a
22-tool allowlist reproduced by the dotfiles repo's `hermes` install step; Claude Code uses
the official Linear plugin. The first issue captured through this flow was the idea of an own
tracker, parked with its own reopen condition.

## Related

- [backlog-and-roadmap.md](./backlog-and-roadmap.md) — the GitHub-issues capture this
  supersedes for adopting repositories; still the path where outsiders file issues.
- [design-workflow.md](./design-workflow.md) — what a groomed issue becomes when design starts;
  its "already promised?" check reads Linear on adopting repositories.
- [feature-docs-as-memory.md](./feature-docs-as-memory.md) — the per-feature doc the
  technical spec grows into once shipped.
- [adr-pattern.md](./adr-pattern.md) — the durable decisions that stay in Git.
- [cross-machine-sessions.md](./cross-machine-sessions.md) — `continue <ID>` is the
  tracker-side counterpart of parking a session in the repo.
