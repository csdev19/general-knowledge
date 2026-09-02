# Plan → Backlog: deliverables executable in parallel

_How to turn an approved implementation plan into self-sufficient backlog documents that several runner agents can execute in parallel — and why the backlog, not the plan, is the execution source._

It extends the [specs + plans workflow](./specs-and-plans-workflow.md): a conversion is inserted
between **plan** and **ship**. The plan (a single linear document) becomes an **epic** + **one doc
per deliverable** inside the documentation app's `backlog/`, following the
[backlog pattern](./backlog-pattern.md).

```
brainstorm → spec → plan → [CONVERSION] → backlog epic + P1..PN → runners in parallel → ship
                              plan becomes superseded ──────┘
```

## Why (do not lose sight of this)

1. **A plan is linear; execution does not have to be.** The plan orders tasks for a single
   executor. On conversion, the real dependencies are derived from the **Interfaces** blocks
   (what each task consumes), not from the lane diagram — which tends to over-serialize. Typical
   result: 3 parallel lanes where the plan said "serial".
2. **A runner only sees its own doc.** Subagents execute with fresh, limited context. That is why
   each deliverable doc is **self-sufficient**: the complete code copied from the plan (never "see
   the plan"), verifiable preconditions with a STOP-and-report rule, commands with expected
   output, a commit step and a Definition of Done. If a doc requires opening another document to
   be executed, the conversion failed.
3. **A single execution source.** After the conversion, the plan is marked
   **superseded — do not execute from there**. Executing from two documents that can diverge is
   how steps get lost. The state lives in the backlog map.
4. **Parallelism only from disjoint files.** Two deliverables run in parallel only if their file
   sets do not touch. "Conceptually separate" does not justify parallelism; a file collision means
   serial.

## The skill that automates it

This pattern lives as a project skill in `.claude/skills/plan-to-backlog/SKILL.md`
(in `monorepo-template` and in the projects derived from it). The skill fixes the output inventory
(epic + P-docs + index rows), the pre-decided conversion rules, and the verification (build the
docs app before committing).

**Evidence that it is worth it** (TDD test of the skill, 2026-07-21, language-cards):
an agent *without* the skill achieved a correct conversion — the pattern is discoverable from the
repo's exemplars — but spent ~100k tokens and 30 tool uses rediscovering conventions, and made
judgment calls on the fly. With the skill: structural compliance with 1 tool use and ~⅓ of the
tokens, and the judgment calls were already fixed. The skill does not fix mistakes: it **removes
rediscovery and variance**.

## Pre-made decisions (do not re-litigate on every conversion)

| Decision | Rule |
| --- | --- |
| Language | Everything in English — prose and status/effort tokens (`🔵 Proposed`, `Low/Medium/High`); see [english-only](./english-only.md) |
| Dependencies | Derived from the plan's **Interfaces** blocks, not from the lane diagram |
| Parallelism | Only from disjoint file sets, justified explicitly in the epic |
| Branches | Integration branch named in the plan; parallel runs on `feat/<epic>-p<N>`, merged back |
| Statuses | 🔵 Proposed → 🟡 In progress → 🟢 Done; the epic moves to 🟢 Ready to validate when every P-doc is 🟢 |
| Index | Rows inserted additively; other epics' statuses are updated by their own work |

## Real exemplars

- `language-cards` → `apps/documentation/src/content/docs/backlog/`: the epics
  `furigana-tokens` (E1–E7, executed) and `particle-sound-tap-explain` (P1–P9) — the second one
  generated with the skill.
