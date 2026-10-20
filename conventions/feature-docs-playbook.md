# Playbook: adopting feature docs, and judging whether they scale

**Feature docs are an experiment, started on 2026-10-19.** The reasoning for them is in
[feature-docs-as-memory.md](./feature-docs-as-memory.md). This page is the procedure for adding
them to a repository, the guardrails that keep them from growing into a document that gets in the
way, and how a trial is judged. The trial ends with a decision either way: keep the docs, or delete
them.

## When to use

Use it when agents in a repository keep rereading code to recover facts that are already settled:
which file owns a flow, which decision applies, which trap bit the last person. It also applies
when a log file has started to stand in for that knowledge (see the progress-log anti-pattern in
the convention page).

Do not use it to replace ADRs, specs or PR bodies. Each answers a different question:

| Question                                         | Answered by                                    |
| ------------------------------------------------ | ---------------------------------------------- |
| How does this feature work today, and where?     | The feature doc                                |
| Why this and not something else?                 | The ADR, linked from the feature doc           |
| What was planned before it was built?            | The spec, frozen once designed                 |
| What did one change check, and leave unverified? | That PR's body                                 |

An ADR is never rewritten; a new decision gets a new ADR. So the current state of a feature is the
sum of several ADRs, their amendments and the code. The feature doc is that sum written once, and
rewritten whenever the feature changes.

## Steps

1. **Pick three features**, the ones where agents reread the most code. Three is enough to see
   whether the docs stay small and current, and few enough to delete cheaply.
2. **Create one flat folder,** `<docs>/features/`, with one file per feature. A feature is what a
   person can do, so one file covers it across every layer it touches.
3. **Generate the index.** The folder's index page renders a list from the files. In Astro
   Starlight, that is a `getCollection("docs")` component filtered on `features/`, plus
   `autogenerate: { directory: "features" }` in the sidebar. Show each doc's `lastVerified`, and
   flag one older than 30 days. Adding a feature is creating its file; nobody edits the index.
4. **Write each doc** from the convention's shape, without the History section (see
   "Deviation under trial" below). Read the code to write it, and check that every path in "Where
   it lives" exists.
5. **Make updating it part of shipping.**
   - **Root instructions:** add a row to the repository's "docs track the code" table: a PR that
     changes a feature rewrites its doc and sets `lastVerified`.
   - **Agent rule:** point the repository's rule for agents at the feature index. Read the index,
     then the one doc, then only the paths it names.
   - **Templates:** update any doc-writing skill or command the repository carries to the flat
     format.
6. **Record the start and the review date** on the feature index page and in the trial table
   below.

## Guardrails: keeping it from getting in the way

These are what kept `progress.md` from staying useful. Each one is a rule the doc-writing skill
enforces.

- **State, not history.** A doc describes `main` now. A changed fact is rewritten, and a fact that
  stopped being true is deleted. History lives in git and in PR bodies.
- **A size budget: 100 lines.** Over budget means too much detail, or two features in one file:
  cut, or split.
- **Links, never copies.** The "Why it is this way" table names a decision and links its ADR. A
  copied rationale is a second version that drifts.
- **Directories, not file lists.** "Where it lives" names directories and entry files, so a
  refactor inside them does not make the doc wrong.
- **Gotchas expire.** A gotcha stays only while it can still bite, cites its PR, and is removed
  when its cause is gone, for example when a pinned version is unpinned.
- **One file per feature, no shared file.** Two PRs on different features never touch the same
  file. A conflict means two PRs changed the same feature, and that one is worth resolving.
- **Status is explicit.** "On main, not deployed" and "Planned, not built" are written down, and
  "Not verified live" is always present.

## Judging the trial

At the review date, measure these. Each has a threshold that decides.

| Measure                    | How                                                                                        | Keep if             |
| -------------------------- | ------------------------------------------------------------------------------------------ | ------------------- |
| Size                       | `wc -l <docs>/features/*.md*`                                                              | Every doc ≤ 100     |
| Staleness                  | For each doc, the PRs merged since its `lastVerified` that touched its paths without editing it | 0–1 per doc         |
| Accuracy                   | Facts found wrong when someone relied on a doc                                             | None that misled a decision |
| Conflicts                  | Merge conflicts in `features/`, excluding two PRs on the same feature                      | None                |
| Use                        | Did sessions start from the doc instead of rereading the code? Owner's judgement, plus agent notes | Yes, for most tasks |

**Keep:** most thresholds hold. Record the decision in an ADR in that repository, extend the docs
to the next features, and mark this page "adopted" with the evidence.

**Delete:** the docs are mostly stale or over budget, or the owner finds they get in the way.
Delete the folder, keep this page with the evidence of why, and do not try again without changing
what failed.

## Deviation under trial: no History section

The convention's shape ends with a History table (date, PR, change). The trial leaves it out. A
table appended on every PR is a small progress log again: it grows without bound, and every PR on
the feature edits its top row. The hypothesis is that `git log -- <path>` and the PR numbers cited
in Gotchas are enough. If the trial shows that readers needed the history, it comes back, capped at
the last five rows.

## Trials

| Started    | Repository          | Scope                                                     | Review     | Result  |
| ---------- | ------------------- | --------------------------------------------------------- | ---------- | ------- |
| 2026-10-19 | invisible-assistant | Account and sign-in, local web stack, release and deploy | 2026-11-19 | Pending |

## Related

- [feature-docs-as-memory.md](./feature-docs-as-memory.md): why feature docs, their shape, and the
  progress-log anti-pattern.
- [adr-pattern.md](./adr-pattern.md): what the feature doc links to instead of copying.
- [verifiable-handoffs.md](./verifiable-handoffs.md): the PR body, where one change's checks
  live.
- [changelog-pattern.md](./changelog-pattern.md): the generated-index lesson this reuses.
