# Agent delegation: the two modes

_When to use an expensive, fast model, when to use cheap long-running models, and — more importantly — why the model tier is almost never what makes delegated work slow._

This doc assumes a setup with an orchestrator agent dispatching subagents
(Claude Code with `Agent`/`Task`, or equivalent). It is written so that an AI with no prior
context can implement the delegation by reading only this.

## The two modes

| | **Mode A — fast, moderately expensive** | **Mode B — slow, cheap** |
| --- | --- | --- |
| Models | Opus 4.8 | Sonnet + Haiku |
| When | You are waiting for the result, interactive session | You are not watching: it runs in the background, on an always-on machine (e.g. a Mac mini) |
| Choice criterion | The cost of human waiting time exceeds the token cost | The work can take a while; nobody is blocked |
| Risk | Spend | A cheap model gets stuck and nobody notices → you need a heartbeat or a final review |

The operating rule: **the mode is chosen by who is waiting, not by the difficulty of the
task.** A hard task nobody is watching goes in mode B with an expensive final review;
a trivial task blocking a person goes in mode A.

## Role split within a run

Either mode uses the same shape. What changes is which model fills each slot.

| Role | What it does | Mode A | Mode B |
| --- | --- | --- | --- |
| **Orchestrator** | Reads the plan, dispatches, reviews between tasks, keeps the ledger | Opus 4.8 | Opus 4.8 (the orchestrator never drops tier: it is the one deciding) |
| **Mechanical implementer** | The plan already carries the full code/YAML/JSON → transcribe + validate | Haiku | Haiku |
| **Implementer with judgment** | Real logic, verifying claims against files, delicate formats | Opus 4.8 | Sonnet |
| **Per-task reviewer** | A bounded diff against its brief | Sonnet | Sonnet |
| **Final branch review** | Decides whether something breaks production | Opus 4.8 | Opus 4.8 (**never** made cheaper) |

The two slots that never get cheaper are the **orchestrator** and the **final branch
review**. Everything else does.

## The model tier is not the bottleneck

Data measured in a real session: 21 subagents, 10 CI configuration tasks
(YAML + JSON + docs), a plan with the full code inline.

| Implementers | Mean duration | Tool calls per task |
| --- | --- | --- |
| Haiku (6 tasks) | 94 s | 10–17 |
| Sonnet (3 tasks) | 75 s | 10–11 |

Haiku took more turns — the known "cheap model uses more turns" effect —
but only mildly. **Moving the 6 mechanical tasks from Haiku to Sonnet would have saved about
2 minutes out of 41.**

Where the time actually went:

```
Total subagent time ≈ 41 min

  Final branch review (1 Opus agent) ......... 11 min   ← the slowest of them all
  9 per-task reviewers (Sonnet) .............. 13 min
  9 implementers (Haiku + Sonnet) ............ 13 min
  Fixers ......................................4 min
```

The conclusion, and the only thing worth remembering from this doc: **the expensive final
review took longer than the six cheap implementers combined.** The slowness came from the
_architecture of the process_ — 18 sequential dispatches — not from the model tier.

⚠️ A single session's sample (n=21) over configuration work. Do not take it as law:
on tasks with more reasoning the gap between tiers widens. What does generalize
is the order of magnitude: **structure dominates tier.**

## The rules that actually cut time

In order of measured impact:

1. **Group sibling tasks into a single dispatch.** In the measured session, four tasks
   produced four nearly identical YAML files and were dispatched separately: 8
   dispatches (4 implementers + 4 reviewers) where 2 would have been enough. Cost of the
   mistake: ~10 minutes, the largest saving available. **If two consecutive tasks touch
   files of the same type with the same shape, they are one task.**
2. **Run the reviewers in the background and in parallel.** A reviewer does not block the
   next implementer: it only reads an already-committed diff. Sequencing them is pure
   dead time.
3. **Skip the per-task review when the task is pure transcription**, and lean on the
   final branch review + the CI gate. See the next section for when NOT to skip it.
4. **Do not spawn one fixer per finding.** When the final review returns N findings,
   dispatch **one** agent with the complete list. N fixers rebuild context N
   times and re-run suites N times.

## When the per-task review does pay off

Skipping it is not free. In the measured session, an 81-second Sonnet reviewer
found a **Critical** bug that the orchestrator had written into the plan: a PR gate
using `git diff BASE HEAD` (two dots) instead of `BASE...HEAD` (three dots).
With two dots, if the base branch advances after the fork, the diff reports files
the PR never touched — and the gate passes exactly when it should have failed.

Heuristic:

- **Per-task review yes** if the task contains executable logic (shell, conditions,
  computations), claims that must be verified against other files, or anything whose
  failure is silent.
- **Not needed** if the task is writing a file whose full content was already in the
  brief and there is an automatic validator (linter, type-check, `actionlint`).
  The final branch review covers it.

A cheap review that finds a silent failure pays for itself. A cheap review over a
transcription already validated by a linter does not.

## How to dispatch (so context does not explode)

Everything you paste into a dispatch prompt — and everything the subagent prints
back — stays in the orchestrator's context for the rest of the session. Pass the
artifacts **as files**, not as pasted text.

```
.<scratch>/task-N-brief.md     extracted from the plan; the ONLY source of requirements
.<scratch>/task-N-report.md    the subagent writes here; it returns only a summary
.<scratch>/review-<a>..<b>.diff  commits + stat + diff -U10 for the range, in one file
```

A dispatch prompt contains this and nothing more:

1. One line on where this task fits in the project.
2. The path to the brief, presented as "read it first, these are your requirements, with
   the exact values to use literally".
3. Interfaces and decisions from earlier tasks that the brief cannot know about.
4. Your resolution of any ambiguity you noticed in the brief.
5. The path of the report file and what it must contain.

**Never** paste the accumulated history of previous tasks ("state after tasks
1-3") into a later dispatch. A fresh subagent needs its task, its interfaces and the
global constraints. Nothing else.

### Rules for writing a reviewer prompt

- Copy the binding constraints **literally** from the plan: exact values, exact
  formats, declared relationships between components.
- **Do not pre-judge findings.** Never write "do not flag X", "treat it as Minor at
  most", or "the plan chose this". If you think a finding would be a false
  positive, let the reviewer raise it and resolve it yourself afterwards. If the prompt you
  are writing contains "do not flag", stop: you are saving yourself a review round
  at the expense of the review.
- Tell it what it does **not** need to re-run (tests the implementer already ran) and what
  it **must** verify independently against real files.

## Durable progress: the ledger

Conversation memory does not survive a compaction. The most expensive failure
observed in practice is an orchestrator that lost the thread and **re-dispatched
already-completed tasks**.

Keep a progress file (e.g. `.<scratch>/progress.md`), outside git or
git-ignored, and on closing each task append a line:

```
Task N: complete (commits <base7>..<head7>, review clean)
```

Rules:

- On start, read the ledger. Whatever it lists as complete **is** complete: do not
  re-dispatch it, resume at the first unmarked task.
- After a compaction, trust the ledger and `git log` over your own recollection.
  The commits the ledger names exist in git even if you no longer remember creating them.
- Also record the **Minor** findings you decided not to fix at the time, and point the
  final review at that list. A roll-up nobody reads is a silent
  dismissal.

## Cross-cutting lesson: name the unit before quoting the number

This came out of the same session and is not about agents, it is about measuring. It cost two
corrections and one decision taken twice.

We needed to know how often a certain commit shape occurred, to decide whether an automatic
guard was worth it. Three attempts:

1. **"0 occurrences in 674 commits"** — the sample was `git log -200`. But the repo
   committed ~200 times a day, so that window covered **a single day** and was
   generalized to the whole history. → **`git log -N` is not a time window.**
   With an uneven commit rate, N commits says nothing about a date range.
2. **"9 occurrences, 8 within 11 days"** — window corrected (the 674 commits), but it
   counted **commits** when the mechanism being evaluated was a **PR check**. The number
   inverted the recommendation by measuring the wrong thing.
3. **Correct** — merges into the main branch were counted: **2 occurrences in the repo's
   entire history.** The 9 commits from attempt 2 were internal PR commits that did
   touch consumers; nothing was ever broken.

The two rules that remain:

- **State the date range your sample actually covers** before generalizing
  from it.
- **Match the unit of measurement to the mechanism you are evaluating.** A PR-level
  guard is measured per PR, not per commit. A commit-level guard, per commit.

Corollary for agents: when you present a measurement as the justification for a
decision, include the exact command that produced it. It was by reading the command that
both errors were caught.

## Startup checklist

To implement this delegation in a new project:

- [ ] Define which model covers mode A and which models cover mode B, and record it where the
      orchestrator will read it (`CLAUDE.md`, `AGENTS.md` or equivalent).
- [ ] Pin the orchestrator and the final branch review to the expensive model. Non-negotiable.
- [ ] Create the scratch directory for briefs, reports and diffs; git-ignore it.
- [ ] Create the progress ledger and the rule to read it on startup.
- [ ] Before dispatching task 1: review the plan for sibling tasks and
      group them. It is the largest saving and it is only available before you start.
- [ ] Decide, task by task, whether it gets its own review (executable logic / silent
      failure → yes; transcription with a validator → no).
- [ ] Run the project's full gate (type-check, lint/format, tests, build)
      **once, before pushing**, not between tasks.

## See also

- [specs-and-plans-workflow.md](./specs-and-plans-workflow.md) — where the plan this
  delegation executes comes from.
- [plan-to-backlog.md](./plan-to-backlog.md) — turning an approved plan into
  self-sufficient deliverables for agents in parallel.
