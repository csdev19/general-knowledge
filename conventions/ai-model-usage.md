# AI model usage: roles, quotas and fan-out

_Which model does which job, how to spend a capped model quota without wasting it, and
what a reviewer agent needs besides a model. Companion to
[ai-agent-delegation.md](./ai-agent-delegation.md), which covers how to dispatch and
why process structure dominates model tier._

## Model roles

| Role | Model | Why |
| --- | --- | --- |
| **Default and orchestrator** — conversation, exploration, research, first drafts of specs and plans, final branch review | Opus 5.5 | Strong enough for judgment; cheap enough to spend on reading |
| **Refiner** — one pass over a finished draft: tighten a spec, resolve contradictions, challenge decisions | Fable | The most capable and the most constrained quota; spend it on thinking, with bounded evidence instead of free exploration |
| **Executor and code reviewer** — implementation from a brief, per-task review, scans | Sonnet | Long tool loops at low cost; reliable when the brief and the tools carry the rigor |

The rule that follows: **Fable never explores without a bound.** Exploration means opening
files, grepping, reading audit reports and doing web research, and every turn pays to
re-read all of it. Opus produces the draft and a compact package: the draft, the decisions
table, the open questions, and the few files whose exact contents matter. Fable starts from
the package and may ask for more evidence (named files, symbols, questions); code or a
cheaper agent fetches it, at most two rounds, each capped in size.

An earlier version of this page said "Fable never explores" with no exception. Two
adversarial reviews rejected it: a refiner that cannot check a claim against the
repository writes confident specs that name APIs or paths that do not exist, and a wrong
spec costs a whole implementation run. The bound, not the ban, is what saves the quota.

The model names change; the roles do not. When a new model arrives, reassign the table
rather than adding a row.

## When parallel agents pay, and when they don't

Parallel is not the problem. Parallel on the wrong model, or on the wrong kind of work, is.

**Parallel pays when** the work items are independent, the model has quota to spare, and a
lost run is cheap. Five audits on Sonnet, one per surface, each writing its own report:
fast, cheap, nothing cut off.

**Parallel wastes when three things combine:**

1. **A small quota.** N agents consume N times as fast; the cap arrives in minutes.
2. **A cap cuts every agent in flight at once.** Two at a time loses at most two runs; seven
   at a time loses seven. Runs that have not started cost nothing.
3. **Exploratory work.** Agents that each read the same inputs (reports, codebase, ADRs,
   web) pay for the same reading N times.

In one line: parallelize what is cheap and independent, serialize what is expensive or
shares the same reading.

### Concurrency defaults

| Model | Agents at once | Condition |
| --- | --- | --- |
| Sonnet | 5–6 | independent items, each with its own output file |
| Opus | 3–4 | drafts and research |
| Fable | 2 | refinement only, from a compact package |

**Shared reading goes first, once.** When several agents need the same material, one agent
reads it and writes a digest file; the others read the digest. A first run on any new kind
of task is a measurement: dispatch one, read its cost, then decide how many more fit.

### The pipeline for design work

1. **Opus drafts, in parallel.** Each agent explores its own area and leaves a draft plus a
   package: decisions, open questions, and the files whose exact contents matter.
2. **Fable refines, two at a time.** Its input is the package only, never the repository.
   One pass to challenge decisions, find contradictions and tighten the text.
3. **Sonnet implements and reviews, in parallel**, in worktrees, after a collision check
   against open pull requests.
4. **Short follow-ups (recording owner answers, small revisions) go to Opus directly.**
   They do not need a refiner.

Projection, not yet measured: a refinement pass reads the package (tens of thousands of
tokens) instead of the full exploration (millions), so Fable's share of a design session
drops by an order of magnitude.

## Where the tokens actually go

In an agent run, the input is not the prompt. Every turn re-sends the whole accumulated
context: everything the agent has read, plus every tool result. Output is a rounding error
next to it. Two consequences:

- **Cost scales with what the agent reads × how many turns it takes**, not with how much
  it writes. Twenty turns over a 120k-token context is about 2.4M tokens read.
- **The four buckets are priced differently.** On Anthropic's API, cache reads cost 0.1×
  the base input price, 5-minute cache writes 1.25× and 1-hour writes 2×; output
  (including thinking) is 5× input. Millions of cached reads can cost less than a few
  hundred thousand cache writes. How a subscription's usage meter weighs each bucket is
  not published: calibrate it by logging each run's usage per bucket next to the usage
  readings, before trusting "context × turns" as a cost model.
- **Reasoning is output.** A high effort level on a strong model is not free even with no
  tools; what makes a tool-less pass cheap is that it is one or two turns.
- **Prompt caching makes re-reading cheap only while the cache is alive.** When a run
  pauses past the cache lifetime (for example, waiting out a rate limit), the next turn
  re-writes the entire context at full price. Resuming is not free; it is close to
  starting over.

## Protocol: before a fan-out on a capped model

A fan-out is several agents dispatched at once. Before one on any model with a usage cap:

1. **Estimate the cost per agent from a measured run**, not a guess: turns × context size
   for a similar task (see the measured example below). If no run exists, dispatch one
   agent first and measure it.
2. **Check the remaining window** with the client's usage view (`/usage` in Claude Code).
   Note when it resets and whether unused quota expires at the reset. Quota that expires
   is free to spend; quota cut off mid-run is not, because the run is lost.
3. **Cap concurrency on the expensive model at two or three.** Agents in parallel hit the
   cap together, and a cap that lands mid-run wastes every run in flight. Sequential waves
   lose minutes; parallel waves can lose everything.
4. **Check for collisions before dispatching fixes.** List open pull requests and branches
   that touch the same paths (`gh pr list` plus each PR's files). An audit run against the
   main branch cannot see work already in flight, and a fix agent will happily duplicate
   it.
5. **Write outputs somewhere durable from the start.** A session scratch directory can be
   wiped when the session ends. Reports another step depends on go to a gitignored folder
   in the repository, and progress goes to a ledger (see the delegation page).

`cs-check-budget` is the procedure for steps 1–2 when the run is large enough to matter.

## Protocol: when an agent hits a usage limit

1. **Do not resume everything at once after the reset.** That is the same simultaneous
   fan-out that hit the limit, and the cache has expired, so it now costs more.
2. **Look at what the agent left before deciding.** Check its worktree, its commits and
   its last message. An agent that had only read (no files written) has nothing to resume.
   A fresh agent with a narrow brief costs less than restoring a context full of
   exploration.
3. **Resume only agents that were mid-write**, one or two at a time, on the model the
   user picks for the continuation.
4. **Removing an agent's worktree ends its resumability.** Follow-ups to a finished agent
   (owner answers, small revisions) go to a new agent with a brief that points to the
   artifact: the spec path, the PR and the decisions. Do not ask the new agent to re-read
   the original inputs.

## Reviewers need tools, not a bigger model

A Sonnet reviewer is as good as the deterministic checks it runs before it judges. Give
every review agent:

- **The diff as a file and the brief it is judged against.** Never a summary of either.
- **The project's gate first**: type-check, lint, format, tests (`verify` or equivalent),
  with what it covers stated.
- **Domain scanners, by what the change touches:**

| Change touches | Tool | Status here |
| --- | --- | --- |
| Secrets and git history | gitleaks, trufflehog | used; clean result was trustworthy |
| Dependencies | the package manager's audit (`bun audit`, `npm audit`), osv-scanner | `bun audit` used; osv-scanner not yet tried |
| CI workflows | actionlint (with shellcheck) | used; catches script injection and expression errors |
| Application code patterns | semgrep with a security ruleset | not yet tried |
| Electron apps | Electron's security checklist; Electronegativity | checklist used by hand; tool not yet tried |

- **A per-surface checklist** (desktop, API, web, CI and infra, database). One reviewer per
  surface beats one reviewer for everything: each one's context stays small and specific.

The model's judgment is for what tools cannot see: authorization logic, data flow,
whether the fix matches the intent. A reviewer that skips the tools spends tokens
re-deriving what a linter would have printed.

### Route reviews by risk, not by rejection count

Escalating to a stronger reviewer after the cheap one asks for changes twice selects the
wrong cases: repeated rejections measure friction (a missing test, a rename), while the
dangerous change is the one the cheap reviewer approves at once with an invariant broken in
a file it never read. Decide the reviewer **before** the first review, from risk computed by
code: sensitive paths (auth, permissions, payments, deletion, migrations, schema), fan-in of
changed exports, contract changes, config and CI, new dependencies. A change the classifier
cannot place counts as high risk. Give the strong reviewer context assembled by code (the
callers of every changed export, the full text of the decisions that mention the touched
modules) and bounded evidence requests.

### AI review in continuous integration

When agents open many PRs, review is a gate in CI, not a conversation:

- Run it only after the required checks are green; a red PR goes back to its author
  without spending review tokens.
- Report the verdict on the exact head commit (a commit status or check) so a later push
  voids it, the same rule as human QA approvals.
- Batch every finding into one repair round; the second review only verifies those fixes.
  Persistent disagreement goes to a human.
- A person merges every PR, whatever its risk (agents never merge; decided 2026-10-28). Risk sets
  how deep the review goes, not who merges.

## Open question: how much better is a Fable review than a Sonnet review?

Not measured yet. Expectations to test, not facts:

- **Deterministic findings** (lint, types, known-vulnerable dependencies, secrets,
  workflow injection) should not differ: the tools find them, and either model reports
  them.
- **Pattern findings with local evidence** (a missing ownership filter, an unvalidated
  path, a permissive config) are where Sonnet with a good checklist is expected to be
  close.
- **Cross-file and intent findings** (a fix that is correct locally but breaks an invariant
  elsewhere; a design that contradicts an ADR; a race between two flows) are where a
  stronger model is expected to pull ahead.
- **False positives** cost review time; a stronger model is expected to produce fewer, but
  that is part of what the experiment measures.

### How to measure it

Run both on the same input and score against ground truth:

1. Pick three to five merged pull requests whose real defects are known: bugs found later,
   review comments that led to fixes, or a defect planted in a copy of a branch.
2. Give each model the identical package: the diff as a file, the brief, the checklist,
   and the gate output already run (so neither spends turns on tools).
3. Score each review: true findings, missed known defects, false positives, by severity.
   Record turns and tokens from the transcripts.
4. Decide per category. The likely outcome is a split, such as Sonnet per task and a
   stronger model for the final branch review of risky surfaces (auth, payments, data
   deletion, migrations).

Record the result as a measured example on this page and update the roles table.

## Model size and code review: external evidence (collected 2026-11-03)

No standard benchmark compares the Haiku, Sonnet and Opus tiers on code review. What exists,
with what each source actually measured. Figures were read on the linked page unless marked.

**Recall against precision across tiers**

- [Bigger Isn't Always Better (arXiv 2606.15689)](https://arxiv.org/abs/2606.15689), June 2026
  preprint. 150 review samples (100 mutation-injected bugs, 50 real bug-fix PRs from eight
  open-source repositories), five models including Haiku 4.5 and Sonnet 4.6. Haiku 4.5: F1 0.365
  against 0.343, 18 % higher recall, 3.2× cheaper per review. Precision favoured Sonnet
  (0.558 against 0.486; figure from a search summary, not checked on the page). Best model F1 0.847
  on synthetic bugs and 0.066 on real PRs; F1 falls from 0.657 on diffs under 10 lines to 0.043
  over 150 lines; near-zero recall on performance bugs for every model.
- [CodeRabbit: Sonnet 5.5 vs Opus 5.5](https://www.coderabbit.ai/blog/sonnet-5-5-model-review),
  vendor post. 13 real open-source PRs, one verified bug each, three-judge majority. Caught:
  Sonnet 5.5 thinking 6/13 (actionable precision 41 %), Opus 5.5 Standard 8/13 (67 %), Opus 5.5
  Max 10/13 (52 %). Their split: Sonnet for the main pass, Opus where a miss is expensive.
- [Bito: Haiku 5.5, Sonnet 5.5 or Opus 5.5 for coding](https://bito.ai/blog/claude-haiku-5-5-vs-sonnet-5-5-vs-opus-5-5/),
  vendor post. Agentic coding, not review. Relevant parts: a planted auth trap (a quoted empty
  token that lets an empty `Authorization` header through) that Haiku 5.5 medium missed and
  Haiku 5.5 high found, for $0.014 against $0.009; Opus leads Haiku by 8–10 points on audit turns;
  Haiku's price doubles above 100K-token prompts, which most of its long-session requests crossed.
- [OpenRouter: Opus vs Sonnet vs Haiku](https://openrouter.ai/blog/insights/claude-opus-vs-sonnet-vs-haiku/)
  and [fast.io](https://fast.io/resources/best-claude-model-for-coding-2026/),
  [haimaker](https://haimaker.ai/blog/claude-opus-vs-sonnet-vs-haiku/): tier guides for coding,
  no review measurement. Not checked page by page.
- [SWE-Review-Bench](https://github.com/lmnst/SWE-Review-Bench): a 20-case pilot of cold review on
  Sonnet 4.5 only; too small to rank tiers.

**Filtering a noisy proposer**

- [Sifting the Noise (arXiv 2601.22952)](https://arxiv.org/abs/2601.22952), ISSTA 2026. LLM agents
  (Aider, OpenHands, SWE-agent) filtering static-analysis false positives. On OWASP Benchmark the
  false-positive rate fell from over 92 % to as low as 6.3 %; SWE-agent with Sonnet 4 flagged 95.5 %
  of false positives at 95.5 % precision on OSS-Fuzz C/C++. Agent frameworks help strong backbones
  and give inconsistent gains with weak ones. Aggressive filtering can also drop real findings;
  no figure in the abstract.
- [Augment: how accurate is AI code review?](https://www.augmentcode.com/guides/ai-code-review-accuracy),
  a vendor guide citing primary sources (not checked one by one): Google's Tricorder kept an
  analyzer only while its effective false-positive rate stayed under 10 %; Coverity reported that
  above 30 % people start ignoring a tool; CR-Bench (arXiv 2603.11078) found a reflection loop
  raised recall from 27 % to 33 % while signal-to-noise fell from 5.1 to 1.95; a CodeRabbit field
  study (arXiv 2607.03316) found 56 % of agentic reviews rejected, false positives the top reason,
  10 % of them from missing visibility into other files; a March 2026 preprint (arXiv 2603.18740)
  found that framing code as bug-free cut vulnerability detection by 16–94 points.

**What the evidence supports, as hypotheses for a review pipeline**

- A cheaper tier at high effort can match or beat a mid tier on recall while losing precision;
  effort matters as much as tier on the cheap model.
- A noisy proposer is usable only behind a filter: published reviews above roughly 10–30 % false
  positives lose their readers. Run deterministic filters first (scope, cited line exists,
  duplicate of an open thread), then a stronger model, batched, on what remains.
- Real PRs and large diffs are far harder than injected bugs; measure on real, held-out PRs.

### Measured example: Wachiman re-reviews (2026-11-02)

Six rebuilt incremental re-reviews of real PRs, two runs per arm, audited blind by a different
vendor's model. Haiku 5.5 high cost $0.07 per re-review against $0.50 for Opus 5.5 high, posted
11 valid findings per run against Opus's 6, but 5 of 17 and 2 of 13 were invalid (Opus 0), most
of them true but pre-existing and outside the push. Sonnet 5.5 medium found almost nothing.
Report: [wachiman-benchmarks `2026-11-02-re-review-models`](https://github.com/niway-dev/wachiman-benchmarks/pull/8).

## Measured example: one remediation session

A security and database audit of a monorepo (Kaipu), five surfaces, then fixes and specs.
Figures come from the subagent transcripts. Output tokens were not reliably recorded per
turn, so only context written to the cache and context read back are shown.

| Run | Model | Turns | Cache written | Cache read | Produced |
| --- | --- | --- | --- | --- | --- |
| 5 audits, one per surface | Sonnet | 8–27 | 0.06–0.12M each | 0.4–2.5M each | 5 reports |
| 6 fix implementers in worktrees | Sonnet | 25–50 | 0.06–0.14M each | 2.0–4.6M each | 1 finished PR; 5 cut off by the session ending |
| 3 spec writers, uninterrupted | Fable | 19–23 | 0.14–0.20M each | 2.1–2.7M each | 3 specs |
| 4 spec writers, cut off and resumed | Fable | 17–36 | 0.42–0.88M each | 1.6–4.9M each | 2 specs; 2 nothing |
| 2 follow-ups after the reset | Fable | 10 | 0.15–0.18M each | 0.3–0.4M each | nothing |

What it showed (revised after two adversarial reviews; see "What the reviews changed" below):

- **Fable spent its quota on reading.** Each spec writer read audit reports, the
  codebase, ADRs and web sources itself (over 2M tokens re-read per spec) before writing
  about 500 lines.
- **Resuming after the wait cost 3–5× the cache writes of an uninterrupted run**,
  because the cache had expired.
- **Seven parallel Fable agents, then four resumed together, hit the cap twice.** Four of
  nine Fable runs produced nothing.
- **Session scratch was wiped when the session ended.** The five audit reports were
  recovered only by extracting them from the agents' transcripts.
- **The audit missed two open PRs** that already implemented one of its fixes. A whole
  ticket turned out to be a duplicate.

What the reviews changed. Priced by bucket, an uninterrupted Fable spec run splits
roughly evenly between cache reads, cache writes and output; in the interrupted runs,
cache writes dominate. So the likely order of causes is: **parallelism** (seven runs hit the
cap together), **runs cut mid-flight** (four produced nothing), **resuming with a cold
cache**, and only then **exploration**. The table does not separate model, task,
interruption and parallelism, so this order is inference until per-run usage is logged.
The cheapest fixes are the fan-out and usage-limit protocols above, not a ban on reading.
A second lesson: in an agent pipeline the implementer, not the reviewer or the refiner,
usually spends the most; measure it before optimizing the others.

What it would have looked like under this page's protocol (a projection, not measured):
Opus writes the five spec drafts from the audit reports. Fable refines them two at a time,
each from a compact package. The fix fan-out runs only after a collision check, and reports
go to a gitignored folder.

## Related

- [ai-agent-delegation.md](./ai-agent-delegation.md): dispatch mechanics, the ledger,
  when per-task review pays. Its model names predate this page; the roles here supersede
  them.
- [evidence-driven-optimization.md](./evidence-driven-optimization.md): the reasoning
  behind `cs-check-budget`.
- [lesson-distillation.md](./lesson-distillation.md): how this page was produced from a
  session.
