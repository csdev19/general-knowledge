# Verifiable handoffs

**Finished work is handed back with the means to check it, not with a summary
to believe.** A pull request body, a "task done" message and an end-of-session
report are all the same artifact: a handoff. Its job is to let the reader
decide whether to trust the work.

The executable version of this convention is the
[`write-handoff` skill](https://github.com/niway-dev/skills/tree/main/skills/write-handoff).
This page is the reasoning; the skill is the procedure.

## The rule

A handoff is judged by one question: **could the reader reproduce every claim
it makes?** If not, it is a request for trust, and trust does not scale past
the first surprise.

That reframes what belongs in it. Not "what I did" — the diff already says
that — but "here is how you check me", with everything else as context.

## The three failures it prevents

| Failure                                | What it looks like                                                                 | Why it costs                                                                                                                                           |
| -------------------------------------- | ---------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Unverifiable claim**                 | "Tests pass." No command, no exit code, no counts.                                 | The reader cannot tell a real green from a misread one. A pipe like `test \| tail -5` reports the exit status of `tail`, so a failing run looks clean. |
| **Buried behaviour change**            | The architecture story leads; the removed fallback is in paragraph six, or absent. | The reader finds it by hitting it, in the middle of something else.                                                                                    |
| **File list instead of a review plan** | Fifteen changed files, no word on what to do with them.                            | The triage that was the writer's job is handed to the reader.                                                                                          |

## Required sections

| Section               | Answers                                                                                       |
| --------------------- | --------------------------------------------------------------------------------------------- |
| What this does        | What the work is, in the reader's terms                                                       |
| Why                   | The problem it solves — for a defect, the concrete input and wrong output, never the category |
| What changes for you  | Every observable behaviour change: before, after, and what to do instead                      |
| Verify it yourself    | Setup, numbered steps, expected result, and the failure mode — ordered by risk                |
| What I did not verify | Paths not exercised, things asserted from reading rather than running, known flakiness        |

"Verify it yourself" is the section the rest exists for. The others are
context that makes it followable.

## Three rules that carry most of the value

**Capture the exit code.** Never claim a check passed without having read its
status directly. Redirect to a file and echo `$?`; never pipe the command
whose status you are about to report.

**Verify against a clean checkout.** When the working tree holds anything
uncommitted, a local pass proves nothing about what CI will build. A
`git worktree` is the cheap way to check the branch as it actually stands.

**Reading a file is a verification step.** When the thing to check is a
judgement call — a contract, a refusal, a default — name the file and the
question to ask of it, as a numbered step. That replaces the file list
entirely: the diff already enumerates, and enumeration is not guidance.

## What does not belong

- A description of the diff. The reader has the diff; what they lack is the
  judgement that produced it.
- A softened failure. "Mostly passing" is not a status.
- Padding. A handoff nobody finishes reading verifies nothing.

## Related

- [english-only](./english-only.md) — the handoff artifact is English whatever
  language the conversation is in.
- [specs-and-plans-workflow](./specs-and-plans-workflow.md) — what precedes
  the work this hands back.
- [ci-cd-pipeline-strategy](./ci-cd-pipeline-strategy.md) — the `verify` gate
  whose exit code a handoff quotes.
