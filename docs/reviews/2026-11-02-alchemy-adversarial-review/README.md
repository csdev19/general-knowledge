# Adversarial review: Alchemy vs wrangler (2026-11-02)

Raw material behind [infra/alchemy-evaluation.md](../../../infra/alchemy-evaluation.md).
Kept verbatim so the reasoning can be audited; the evaluation page is the conclusion.

| File | Author | Role |
| --- | --- | --- |
| [00-context-opus.md](./00-context-opus.md) | Claude Opus 5.5 (web research sub-step on Sonnet 5.5) | Neutral context package: current pipeline, prior decisions, Alchemy state, numbered facts F1–F22 and uncertainties U1–U12 |
| [01-for-fable.md](./01-for-fable.md) | Claude Fable 5.1, effort medium | Advocate FOR adoption |
| [02-against-astra.md](./02-against-astra.md) | GPT-6 Astra, reasoning effort low | Advocate AGAINST adoption |

Both advocates worked only from the context package plus at most eight named extra reads.
Repository paths in the reports use `K/` for `kaipu-record-monorepo`, `H/` for this hub and
`T/` for `monorepo-template`, all under `~/Documents/Workspace/Personal/`. Snapshot: kaipu
`c01796bb`, hub `93134fe`.
