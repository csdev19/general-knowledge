# Evidence-driven optimization

**You cannot optimize what you cannot measure, and a measurement nobody
projected to the limits is not yet a decision.** Three failures recur
whenever a working system has to get faster, cheaper or smaller: changing
code on a hunch, measuring something other than what the limit measures, and
spending something scarce on numbers that were never projected to the
workload the system will actually face. This convention names the three
artifacts that prevent them and the order in which they are produced.

The executable versions of this convention are three skills in the
[skills repo](https://github.com/niway-dev/skills):
[`cs-capture-evidence`](https://github.com/niway-dev/skills/tree/main/skills/cs-capture-evidence),
[`cs-check-budget`](https://github.com/niway-dev/skills/tree/main/skills/cs-check-budget) and
[`cs-optimize-with-evidence`](https://github.com/niway-dev/skills/tree/main/skills/cs-optimize-with-evidence).
This page is the reasoning; the skills are the procedure. For things that
are broken rather than slow, the `diagnose` skill's feedback-loop discipline
applies first.

## Three artifacts, in order

| Artifact | Answers | Produced by |
| --- | --- | --- |
| Evidence file | What is known, how it is known, on which workload, runtime and code version, and what each check verified | `cs-capture-evidence` |
| Budget table | Whether it fits under each published limit, with measured, projected and bounded rows kept apart, which limits no test reached, and with what margin policy | `cs-check-budget` |
| Optimization record | The objective, what changed, what each change bought on the objective's workload, what dominates now | `cs-optimize-with-evidence` |

Each depends on the one before it. A budget row with no evidence behind it
is a guess; an optimization without an objective has no finish line.

## Evidence records provenance, not just numbers

A number is usable when a reader can tell how it is known: observed in this
session, reported by someone or a previous session, or derived from other
entries; on which code version, workload mix and runtime; by which command;
and what the check verified. Those are different claims: that a run
completed, that it stayed under a resource limit, that a sample of answers
matched, that every answer matched an oracle. A capacity probe proves
completion, not correctness.

Material the user supplies is evidence of what was captured, not an
independent measurement; a screenshot with no command behind it is kept as
an incomplete observation with a note on what it can still support, not
discarded. Entries are never erased; a wrong one is marked superseded or
invalid so the reader finds the current evidence from the table instead of
reconstructing it from history. Observations, derived values, hypotheses and
missing information live in separate sections.

When the environment is hard to test — the incident happened once, the page
stayed down, a data sample takes days — the file is the only ground there
is; the analysis says so, keeps every conclusion traceable to an entry, and
prices the gaps with the cheapest proxy for each. That is how "we cannot
test this" becomes a plan instead of an ending.

## A budget keeps measurement, projection and bound apart

Published limits are the test cases the author bothered to write down. A
system that passes its own tests at a fraction of those limits has proven
nothing about them. The budget projects each limit, but a projection is only
as good as its assumptions: the operation mix, whether unit cost grows with
size, periodic work a small run never paid, overhead counted once. Mixed
workloads are budgeted by component. Multiplying a small-run average by the
cap is a projection; it becomes a bound only with an argument, and it
becomes a measurement only when a workload at the cap actually runs.

Metric and limit must match. Process RSS is not guest heap; wall-clock is
not CPU; the measured phase is not the whole run; another runtime is not
this one. A proxy metric is allowed when it is labeled, and it cannot carry
a verdict of "fits" by itself. The margin is a policy chosen per action and
recorded with its reason, not a universal constant.

Four verdicts: fits the evaluated envelope, fits with identified risk, does
not fit, undetermined. "Undetermined" exists so that missing evidence is not
disguised as risk. Reaching each scalar maximum in some test does not cover
every allowed combination; the tested envelope and the remaining gaps are
written down. The verdict informs the decision to spend; authorization stays
with the owner and the project's rules.

## Measure where it runs, and say when you could not

Profilers on a host JIT describe a different machine than an interpreter in
a sandbox, a device, or a constrained tier. When the real runtime is
reachable, timers go inside it, around phases, with the clock's resolution
respected. When it is not, proxy results are labeled with their limits and
never presented as production validation. A baseline that timed out is
censored: it bounds the cost from below and supports no speedup claim until
the fixed version is measured.

## Attribution experiments are evidence, not accounting

A constant-returning stub estimates harness overhead but also removes
output, allocation and state. Variants with one component disabled or
doubled attribute cost, but their differences include interactions —
allocation, garbage collection, caches — and are not exact component costs.
Diagnostic variants return wrong answers by design; they are labeled, kept
apart from correctness-validated results, and never promoted into the
candidate.

## Prioritize by contribution to the objective

Every optimization has an explicit objective: a cap to meet, a cost to
reduce on a defined workload, a latency target, a measured growth problem,
or a question of whether a bottleneck matters. Components are ranked by how
much of the unmet objective they account for over that workload envelope. A
per-query constant paid tens of thousands of times can outweigh an
occasional rebuild; the rebuild can dominate a growth ratio. Neither
"growth first" nor "constants first" is a rule. Size scaling is measured to
check that an improvement stays sufficient as the workload grows; two sizes
give a growth signal, not a complexity class, because fixed overhead, mix,
periodic work, warm-up, garbage collection and noise all move the ratio.

Amortizing, rescheduling, batching and doing less work are different
mechanisms with different effects on total cost in a measured window; a
change names which one it is.

## Correctness checks complement each other

Optimization rewrites the thing the tests were written after. The slow
version being replaced is usually a correct oracle, and a differential fuzz
against it catches cases no hand-written test anticipated — but it shares
any misreading of the contract its author made. Small expectations derived
independently from the contract, invariants, boundary cases and a seeded
bug that confirms the harness catches bugs complete the set. When a
hand-derived value and the oracle disagree, the disagreement is
investigated, not resolved by preference. No single check proves
correctness.

## When to stop

When the objective is met with the margin the owner asked for, on the
workload the objective names. Not when ideas run out, and not when local
numbers look good at the sizes the tests happened to use.
