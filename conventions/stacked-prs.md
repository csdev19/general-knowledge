# Stacked PRs with `gh stack`

> **One-minute summary.** A PR that needs the code of another open PR is a layer of a **native
> GitHub stack**, not an ordinary PR pointed at another branch. Create and repair stacks with
> `gh stack`; confirm membership by reading the PR's `stack` field, because a PR opened with
> `gh pr create --base <other-branch>` looks stacked and is not. Use only the commands in
> [Headless-safe commands](#headless-safe-commands); the others open a prompt or a full-screen
> view and block an agent forever. The owner merges with `gh stack merge`, never `gh pr merge`.
> An agent never merges.

_Reference: GitHub's own agent skill and docs in
[github/gh-stack](https://github.com/github/gh-stack/tree/main/skills/gh-stack) (command
behaviour, exit codes, troubleshooting). This page is our rule set, distilled from it and from
using it. `gh stack <command> --help` is authoritative for flags; this page says which commands we
use and what we conclude from them. Checked 2026-11-15: the command help of `gh-stack` v0.3.0 and
the reference skill at its version 0.2.0._

## When a PR is stacked

- **Stacked:** the PR cannot build, run or be reviewed meaningfully without the code of another
  PR that is still open. It becomes a layer above that PR.
- **Independent:** it can land on the default branch alone. It branches from the default branch,
  even if another PR is open at the same time. Do not stack to "keep things together".
- A stack is **strictly linear**: each layer has one parent and at most one child. Parallel work
  is separate stacks.
- **One stack, one story.** A reviewer walks the layers bottom to top and watches one feature
  being built. Unrelated work gets its own stack.
- Put foundational work at the bottom (a design PR, a schema, a shared type) and code that
  depends on it above. The order cannot be changed non-interactively afterwards, so decide the
  layers before writing code.

## Stacks are not automatic

GitHub does **not** enrol a PR in a stack because its base is another PR's branch. A chain made
with `gh pr create --base <branch>` is an ordinary set of PRs: their `stack` field is `null`,
they merge one at a time, and merging a lower PR into the default branch can leave the one above
merged into a dead base branch. Stacking is a separate object, created by `gh stack init` +
`submit`, or by `gh stack link` for PRs that already exist.

**Never create a PR that depends on another open PR with `gh pr create --base`.** Use the commands
below, then verify.

## Headless-safe commands

`gh stack` checks whether stdout is a terminal. Under a terminal (and some agent harnesses give
one), a bare command opens a prompt or a full-screen view and waits forever. Always pass the
arguments in the second column.

| Intent | Run | Never run bare |
| --- | --- | --- |
| Start a stack | `gh stack init <branch> [<branch>...]` (bottom to top; existing branches are adopted, missing ones created) | `gh stack init` (asks for names) |
| Add a layer | `gh stack add <branch>` from the top branch | `gh stack add` (asks for a name) |
| Open the PRs | `gh stack submit --auto --open` (ready for review; `--auto` alone opens drafts and does not prompt) | `gh stack submit` (asks for a title per PR) |
| Read state | `gh stack view --json` | `gh stack view` (full-screen view) |
| Enrol existing PRs | `gh stack link <pr-or-branch> <pr-or-branch>...` (bottom to top) | — |
| Replay layers after a change | `gh stack rebase` (or `--upstack` from the changed layer) | — |
| Routine refresh | `gh stack sync` | — |
| Move between layers | `gh stack up`, `down`, `top`, `bottom` | `gh stack switch`, `gh stack checkout` with no argument (menus) |
| Merge (owner only) | `gh stack merge <pr-or-stack-number> --yes` | `gh pr merge` |
| Restructure | `gh stack unstack`, then `init` again | `gh stack modify` (full-screen editor only) |

Rules that follow from this table:

- **Name every branch yourself.** Names are used verbatim (`gh stack add refactor/foo` creates
  `refactor/foo`). Follow the repository's own branch-naming convention. `add -m` with no name
  generates a date-and-slug name.
- **Create the stack before writing the code, not after.** Make one commit per concern, in the
  layer that owns it. Never commit a lower layer's concern on the top branch; check out the
  owning layer (`gh stack down`, or `gh stack checkout <branch>`), commit there, then
  `gh stack rebase --upstack` and `gh stack top`.
- **`--auto` titles are mechanical.** A branch with one commit uses its subject and body; a branch
  with several commits gets a humanised branch name. There is no flag for a custom title or body:
  write the real ones afterwards with `gh pr edit`.
- **`--open` makes new and existing PRs ready for review.** Without it new PRs are drafts. Use the
  flag only when the work is meant to be reviewed now.
- **Several remotes:** pass `--remote <name>` to `push`, `submit`, `sync`, `rebase` and `link`, or
  set `git config remote.pushDefault origin` first. `checkout` and `trunk` have no `--remote`
  flag and need the config.
- **`link` needs a checkout of the repository** and has no `-R`/`--repo` flag, but it needs no
  local stack tracking.
- **Conflicts and exit codes.** `rebase` and `sync` exit `3` on a conflict (resolve, `git add`,
  then `gh stack rebase --continue`, or `--abort`). Exit `9` means stacked PRs are unavailable on
  the repository (see [the 404](#stacked-prs-are-not-enabled)). Exit `2` means the branch is not in
  a stack. `view --json` writes only JSON to stdout; status text goes to stderr, so branch on the
  exit code, not on messages.

## What `push` and `sync` do not do

| Command | Does | Does **not** |
| --- | --- | --- |
| `push` | Pushes every active branch in one atomic push with `--force-with-lease` | Create or update any PR; create or change the stack |
| `sync` | Fetches, rebases the stack when the trunk or a layer moved, pushes, refreshes PR state, and links the stack's **existing** open PRs when there are two or more | Open a PR (that is `submit`); remove a PR from a stack; sync when local and remote stacks diverged |
| `submit` | Pushes, creates a PR for each branch without one, fixes bases, creates or updates the stack | Take a custom title or body |

Two traps:

- **`sync` can report success without syncing.** Non-interactively, when the local stack and the
  stack on GitHub diverged (for example a PR was added on github.com), it prints both chains,
  changes nothing and exits `0` with `Sync aborted`. Check for that message, or compare
  `gh stack view --json` before and after. To resolve: keep the remote version
  (`gh stack unstack --local`, then `gh stack checkout <stack-number>`), or keep the local one
  (`gh stack unstack`, `init` with the layers, `submit --auto`). Neither deletes PRs or branches.
- **"Branches synced" is not "Stack synced".** The first means branches were rebased and pushed but
  no stack object was created or updated (fewer than two PRs exist yet).

## Verify membership

After creating or repairing a stack, and again before a handoff, check that **every layer** is in
it. The authoritative check is the `stack` field of the pull request, which is `null` for a PR
that is not a layer:

```bash
# one layer
gh api repos/<owner>/<repo>/pulls/<n> --jq '.stack'
# {"number": 12, "position": 2, "size": 4, ...}  -> member (layer 2 of 4)
# null                                          -> not a layer: repair it
```

Every layer must show the same `stack.number`. From a
checkout, `gh stack view --json` is the second check: `branches[].pr.number` for every layer, no
layer with `needsRebase: true`. Neither check replaces the other: `view` reads the local stack,
the `stack` field reads GitHub.

`gh api repos/<owner>/<repo>/stacks` lists a repository's stacks; an empty list means the feature
is on and no stack exists yet.

## Repair a fake stack

A chain made by hand (`gh pr create --base`), or a PR left out of its stack, is repaired without
closing or recreating anything:

```bash
# from any checkout of the repository, PRs bottom to top
gh stack link <pr-bottom> <pr-2> <pr-3> <pr-top>
```

`link` accepts PR numbers, PR URLs and branch names. It pushes branches given by name, creates
missing PRs with correct bases, corrects a wrong base on an existing PR and creates the stack. It
is **additive only**: it never removes a PR from a stack. To append to a stack that exists, pass
the stack number first: `gh stack link <stack-number> <new-pr>`. A PR that already belongs to a
different stack is rejected.

Because `link` writes no local state, `up`, `down`, `top` and `bottom` do not work on the result.
Run `gh stack checkout <stack-number>` if you later want local tracking. Re-check the `stack` field
of every layer afterwards.

## Changing a layer

When a layer changes, the layers above it are **rebased**, not merged into:

```bash
gh stack checkout <layer>      # the layer that owns the change; commit and test there
gh stack rebase --upstack      # replay every layer above onto it
# run the tests on the top layer, then on the layer on the default branch (below)
gh stack push                  # only now: atomic push of all active branches
```

`rebase` replays each layer in order and detects a merged lower PR (including a squash-merge), so
a merged parent does not produce spurious conflicts. `git rerere` is enabled by `init`, so a
conflict resolved once is replayed on the next layer. A review verdict that is keyed to a layer's
**own** diff (`git patch-id` of `<parent>..<layer>`) survives the rebase when that diff did not
change; a verdict keyed to the whole branch does not.

After any rebase, the merge-ready claim for every changed head is void until it is re-checked: the
full suite runs again on the top layer and on the layer that targets the default branch,
**before** `gh stack push`, and each layer's `git patch-id` is compared with the one at its
verdict. A clean rebase does not prove the suite still
passes, because a lower layer can change what a value means without touching a line the upper layer
changed.

## When the stack sat open: lower layers merged, `push` refused

A stack that stays open for days meets two problems at the next rebase. Both look like "the push
does not work"; neither is about authentication.

**Check which layers already merged, before rebasing.** List the stack's PRs with their state
(`gh pr list --search "<ticket>" --state all --json number,state,baseRefName`). A layer that is
`MERGED` is on the default branch, and a squash-merge gives it new commit ids, so the old layer
branches no longer describe what is on `main`. Rebase the remaining layers onto
`origin/<default>`, starting after the last merged layer's old tip:

```bash
git fetch origin
git rebase --onto origin/<default> <old tip of the last merged layer> <first remaining layer>
# then each upper layer onto the rewritten one, using the old tip of its parent
git rebase --onto <rewritten parent> <old tip of parent> <layer>
```

Record every old tip in a file before the first rebase; `--onto` needs them, and they are gone
once a branch moves. When `gh stack rebase` cannot run (no valid token in a headless session, for
example), this is the same replay done with plain `git`; the stack object on GitHub is untouched,
and `gh stack push` from a terminal that is logged in publishes the result.

Symptoms of skipping the check: the bottom PR stays `DIRTY` or `CONFLICTING` after the "fix", its
head commit on GitHub is still the old one, and the diff against the default branch carries the
merged layers' commits a second time.

**A refused push is usually the pre-push hook.** `gh stack push` runs `git push`, so the
repository's `pre-push` hook runs too, and its failure is reported only as "failed to push some
refs". Reproduce it to read the real message:

```bash
git push --dry-run --force-with-lease origin <top-layer>
```

In a non-login shell (ssh, an agent) the hook can also die with `command not found` because the
toolchain is not on `PATH`; set the same `PATH` a terminal has. What a long-lived stack typically
trips:

- **An ADR number taken on the default branch.** Another PR merged an ADR with the same number
  while the stack was open. A check that compares with `origin/<default>` fails, and the fix is to
  renumber the stack's ADR (file name, title, and every link and comment that names it) in the
  layer that introduced it, not in a loose top commit. If the collision is already on the default
  branch (the merged lower layer carried the old number), the renumber becomes the first commit of
  the remaining stack.
- **Formatting of files resolved by hand.** A conflict resolution can leave a file the formatter
  would change. Run the formatter on the files you touched, in the layer where each first went
  wrong, so every PR passes its own check and not only the top one.

After the rebase, the earlier rule applies: the claim "ready" is void until the checks have run on
the rewritten heads.

## Merging

The owner merges. An agent never merges or enables auto-merge, at any risk level.

```bash
gh stack merge <pr-number> --yes    # that PR and every unmerged PR below it
gh stack merge <stack-number> --yes # every unmerged PR in the stack
```

- **Not `gh pr merge`.** It cannot merge a stack.
- The merge is **all-or-nothing**: if any PR in the set cannot merge (draft, not open, a failing
  required check or rule), none do, and the reason is reported.
- Bypassing branch protection is not supported for stacks.
- Without a method flag the last-used method is reused. With a merge queue on the base branch the
  stack is queued instead, the queue picks the method (a flag is ignored with a warning), and the
  PRs may land in separate groups.
- Afterwards `gh stack sync --prune` removes local branches of merged PRs. Check the default
  branch (`git log origin/<default>`), not the "Merged" badge.

## Stacked PRs are not enabled

`gh stack` or the stacks API can answer `404`, or `submit` can exit `9` with "Stacked PRs are not
enabled", when the repository does have the feature. Check these two causes, in this order, before
concluding it is missing:

1. **The wrong GitHub account.** A machine can be logged in to `gh` with several accounts, and the
   active one may not be the one the repository belongs to. `gh auth status` shows which is
   active. Switch to the account the repository's remote maps to, then retry.
2. **`GH_TOKEN` or `GITHUB_TOKEN` in the environment.** It overrides the logged-in account for
   every `gh` call, including the extension's. Check that it is set without printing it
   (`[ -n "$GH_TOKEN" ] && echo set`), run the command with it removed
   (`env -u GH_TOKEN -u GITHUB_TOKEN gh stack ...`), and retry.

If both are ruled out, the repository may really lack the feature. `gh api
repos/<owner>/<repo>/stacks` returns a list (possibly empty) when it has it. When it does not,
tell the owner; do not fall back to `gh pr create --base`. Under a terminal `submit` offers to
create ordinary unstacked PRs. That is the fake stack this page exists to prevent, and an agent
must not accept it.

## Related

- [development-pipeline.md](./development-pipeline.md): the gate; a stack's contiguous verified
  run from the bottom is what is handed over.
