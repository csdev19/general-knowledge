# Cross-machine sessions

**Code survives a laptop switch because it is committed; the session does
not.** What was asked, what was decided, where the work stopped and what was
still open live in the conversation, and the conversation stays on the
machine. Reconstructing it by hand costs a series of commit-and-PR
instructions per project, and something is lost anyway.

The executable version of this convention is the pair
[`cs-park`](https://github.com/niway-dev/skills/tree/main/skills/cs-park) and [`cs-pickup`](https://github.com/niway-dev/skills/tree/main/skills/cs-pickup). This page is
the reasoning; the skills are the procedure.

## The rule

The repo is the transport and a fixed file is the restart document. Parking
writes `.session/HANDOFF.md` at the repo root, commits it as one disposable
checkpoint, pushes, and opens a draft PR. Picking up reads it, summarises it
in three lines, undoes the checkpoint, and waits for the go.

The file is a **restart document, not a transcript**: goal, mode, where we
stopped, what is decided, what is open, git state, and what the agent needs
to start again without asking (the original request verbatim, the files in
play, the check command). Anything that only helps re-read the conversation
is left out.

## Why these choices

**A committed file, not the PR body.** Pure-idea work has no code to open a
PR for, and a closed PR takes its context with it. The file works offline and
lands with `git pull`; the PR body is a copy for reading without checking
out.

**One fixed commit message.** `wip: session checkpoint` is what pickup
recognises. Only that commit is ever rewritten; the user's real commits are
never touched, and the rewrite is guarded by `--force-with-lease`, which
fails instead of overwriting if two sessions touched one branch.

**Never on `main`.** Parking on the default branch creates a `wip/`
branch first. This is the same rule as every other change.

**Restoring is explicit.** A command, not a session hook: the user reads the
summary before the agent touches anything, and the skills stay portable
across runtimes. Discarding is a sentence in chat, followed by closing the PR
and deleting the branch, with confirmation.

## Non-goals

Syncing the assistant's own conversation history; parallel work on one branch
from two machines; a central index of parked sessions, which
`gh pr list --draft --author @me` already answers.

## Related

- [verifiable-handoffs](./verifiable-handoffs.md): the PR that closes the
  parked branch is handed back like any other work.
