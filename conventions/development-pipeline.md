# Development pipeline: from spec to a merge-ready PR

_How agents carry work from an approved spec to a pull request that is ready for a person to merge,
without interrupting the owner, who reviews once at the end and merges. **Agents never merge.**
This page defines the flow and **the gate**:
skills follow it in live sessions, and an unattended runner enforces it in code where it runs.
Where the runner's code is stricter than this page, the code wins and this page should be fixed._

Status: adopted 2026-10-27; the skills and runner changes that apply it are being built.
Corrected 2026-10-28: an earlier version let low-risk PRs merge without the owner. The owner decided
that agents never merge, whatever the risk. The unattended machine cannot run live end-to-end checks
for most products (mobile and desktop apps, memory limits), so green CI is not proof enough to land
code with nobody looking. The decision is the development-workflow ADR "Agents never merge"
(niway-dev/development-workflow (ADR-0020)).
Design: [niway-dev/skills spec](https://github.com/niway-dev/skills/blob/main/docs/specs/2026-10-27-development-pipeline-design.md)
and its ADRs 0001–0006.
Skills that apply it: [`cs-autopilot`](https://github.com/niway-dev/skills/tree/main/skills/cs-autopilot),
[`cs-review-loop`](https://github.com/niway-dev/skills/tree/main/skills/cs-review-loop),
[`cs-babysit`](https://github.com/niway-dev/skills/tree/main/skills/cs-babysit),
[`cs-tracker`](https://github.com/niway-dev/skills/tree/main/skills/cs-tracker), and each project's
`verify` skill.

## Why

An agent that writes good code still leaves a person as the bottleneck if every PR needs that
person to chase CI, answer bots and decide whether the change is safe. Two failure modes come with
fixing that carelessly: an agent that stops to ask questions and blocks the owner's terminal, and
an agent that declares its own work safe and merges it. The pipeline removes the first by never
asking mid-run, and the second by never letting an agent merge: only a person does, and only code
that did not write the change decides its risk.

## The flow

| Stage | What happens |
|---|---|
| 0. Capture | An idea becomes a tracker issue with acceptance criteria, target repo and size |
| 1. Design | Options, spec and ADRs ([design-workflow](./design-workflow.md)); this hub is consulted before deciding |
| 2. Contract | The approved spec is committed on the issue's branch; it is what the code is checked against |
| 3. Build | One **verifiable unit** per acceptance criterion: the failing test or reproduction first, then the code, the check and the commit |
| 4. Prove | Each criterion gets a proof row, on the real surface when it is UI (see Verification) |
| 5. Review | A review loop with a budget in rounds (see Review loop) |
| 6. PR | Why, what changed, the proof list, the decision log ([verifiable-handoffs](./verifiable-handoffs.md)) |
| 7. Babysit | CI, bots and review threads until merge-ready ([review-feedback](./review-feedback.md)) |
| 8. Hand off | Every PR goes to the owner with the final report; the owner merges |

Work happens in a git worktree inside the repository and in subagents; the main session
coordinates. No step needs a separate plan document: the spec's criteria are the plan. Work that
spans several PRs is split into tracker sub-issues, one per PR, each with its own criteria.

## Two entry points, the same playbooks

A **playbook** is a short ordered list of steps, each naming the skill to load when it arrives. The
agent copies the steps into its todo list verbatim, so the sequence lives in the list, not in the
agent's memory. A step not taken stays in the list as `skip: <reason>`.

- **Live (router).** The agent classifies the request (feature, bug fix, refactor, babysit, stack)
  and says so in its first line. It asks the rigor questions **once, at the start**: the end state
  as a checkable predicate, what must not change, what counts as proof. Then it never asks again.
  A question whose answer can be observed by running something is answered by running it.
- **Queue (label).** The runner picks the playbook from the issue's type label. Questions are
  settled before the issue is ready; an issue that lacks them is not dispatched.

**No questions mid-run.** A blocker is written on the issue and the PR; that unit stops and the
rest of the run continues. Every run ends with a **decision log**: each decision, the options
considered, the one taken, so the owner can ask "why this" at review time and change it.

## The gate

Ready for a person to merge (**merge-ready**) means, on the current head:

1. every acceptance criterion has a proof row with evidence;
2. required checks are green, with CI classified before any retry;
3. the review loop ended with no open blocker or should-fix finding;
4. the review verdict was given for **this** head: a new push or a rebase voids it (compare
   `git patch-id` at the verdict commit with the current head);
5. the branch is up to date with its base and the forge reports it mergeable.

**An agent never merges, at any risk level.** A PR that meets the gate stays open and goes to the
owner with the final report; the owner merges it. The verifier's risk sets how deep the review goes,
not who merges. In a stack, only the contiguous verified run from the bottom is handed over, and the
owner merges it bottom-up, one PR at a time.

## The proof list

Every PR body has this section, one row per acceptance criterion of the issue:

```markdown
## Proof

| Criterion | What was done | How to check it | Evidence |
|---|---|---|---|
| <copied from the issue> | <one line> | <command or steps> | <output, screenshot or video link> |
```

The runner refuses review and hand-off while a criterion has no row or a row has no evidence, and
spends one repair round to fill it. Judging whether the evidence convinces stays with the reviewer
and the owner.

## Risk

The **verifier** classifies risk, by code, from the diff: sensitive paths and words, contracts,
config and CI, dependencies, deleted tests, widely imported exports; anything unclassifiable is
high. The author may raise the risk with a reason; nothing the author writes can lower it. The
author wants green and cannot see what it did not know it touched.

The verifier's PR comment opens with a callout whose label is always `Risk: <level>`:

```markdown
> [!CAUTION]
> **Risk: high.** Touches `src/auth/session.ts` (sensitive word: session).
```

Low risk uses `[!NOTE]` and `**Risk: low**`. Levels are `low` and `high` while the review rules are
binary; a new level is added only when it changes something (reviewer model, rounds, second
provider). Risk decides how much review a PR gets, never who merges.

## Review loop

- Round 1 reviews the whole diff; later rounds only open findings and the code changed since.
- Budget: **3 rounds, 5 for high risk.** Check the usage window before each round. A subscription
  runs out of usage windows, not dollars, so rounds are the unit the loop controls.
- Severity: **blocker** (the intended case is wrong), **should** (a real bug callers will hit),
  **nit** (everything else: other paths, rare cases, small costs, extra compatibility).
- Stop when only nits remain. Decline nits in one line and list them as follow-ups. Prefer the fix
  that deletes or narrows code; review loops grow PRs one cheap fix at a time.
- Keep a ledger per finding: id, severity, outcome (fixed with commit, disagreed with evidence,
  declined).
- Budget end, a timeout or no reply is **never** approval: the PR goes to the owner with both sides.
- Models: a cheaper reviewer for low risk, a stronger one for high risk, and for high risk one more
  review on a **different provider** after the fixes; different models miss different defects.

## Babysitting

- Order: conflicts, then review threads, then CI. Batch known fixes into one push.
- A flaky job earns **one** fresh build. An identical second failure means it was not a flake: read
  the logs.
- A failure in code the diff never touches means a stale base (`git merge-base --is-ancestor`), not
  a flake: rebase instead of retrying.
- One watcher per PR, never a hand-written polling loop. Act only on checks and comments newer than
  the last push.
- Babysitting ends at a report, `merge-ready` or `blocked`. It never merges or enables auto-merge.

## Verification on the real surface

Each project keeps a `verify` skill: a map of scenarios, each a command that drives the real app and
writes screenshots and a video into a folder the proof list links. Use the drivers' own recording,
which runs unattended and needs no screen-recording permission:

| Platform | Driver | Screenshot | Video |
|---|---|---|---|
| Web | Playwright | `page.screenshot` | `recordVideo` |
| Electron | Playwright `_electron.launch` | same | same |
| Expo / React Native | Maestro | `takeScreenshot` | `startRecording` / `stopRecording` |
| iOS simulator | `xcrun simctl` | `io booted screenshot` | `io booted recordVideo` |
| Android emulator | `adb` | `exec-out screencap` | `screenrecord` |
| Tauri | `tauri-driver` (WebDriver) on Linux and Windows; macOS unconfirmed | WebDriver | OS recorder |
| Swift native | XCUITest | `XCTAttachment` | simulator recording |

## How we know it works

The runner already records every lifecycle event. The numbers that judge this pipeline: repair
rounds per PR, attempts ending blocked, proof rows missing at first review, and escaped defects per
twenty merged PRs (defects found after the owner merged). Neither reference stack publishes such
numbers; these are ours to collect.

## Prior art

- [tstack](https://github.com/t3dotgg/tstack): `full-send` / `full-stop` (autonomy as a permission
  the phrase grants, a merge gate), `sol-loop` (budgeted review loop), `babysit-pr`.
- [pstack](https://github.com/cursor/plugins/blob/main/pstack/README.md): `poteto-mode` (one mode
  routing to playbooks), the babysit and shipping playbooks, `interrogate`, `create-verification-skill`.

## Related

- [design-workflow.md](./design-workflow.md): stages 1–2.
- [verifiable-handoffs.md](./verifiable-handoffs.md): the PR body and the proof list.
- [review-feedback.md](./review-feedback.md): treating each review comment as a claim to verify.
- [audit-briefs.md](./audit-briefs.md): briefing the reviewer.
- [ai-model-usage.md](./ai-model-usage.md): which model plays which role.
- [ci-cd-pipeline-strategy.md](./ci-cd-pipeline-strategy.md): the CI gate the PR must pass.
