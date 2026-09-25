# GitHub Actions cache: what it costs, and why it fills up

The Actions cache is free — it is billed neither as minutes nor as the storage you pay for
(that is _artifacts_). What it has instead is a **hard 10 GB per repository**, and a
lifecycle that surprises people. A monorepo with a steady PR rate can sit at 87 % of that
ceiling without anyone doing anything wrong.

## The three rules that matter

1. **A cache is scoped to the ref that created it.** A `pull_request` run uses
   `refs/pull/<n>/merge` — **not** the head branch.
2. **Entries are immutable.** Writing a key that already exists is skipped, not replaced.
3. **Eviction**: entries unused for 7 days are removed, and at 10 GB GitHub evicts by
   least-recent-use.

Each of those has a consequence people discover the expensive way.

## Rule 1: deleting the branch on merge does not delete the PR's caches

Merging with `--delete-branch` removes the head branch and anything scoped to
`refs/heads/<branch>`. The caches a PR's CI created live on the **merge ref**, which
outlives the branch. So every merged PR leaves its caches behind.

Measured on a real repository: a bun install cache of ~340 MB **per PR**, and 21 closed
PRs holding **8.1 GB** of the 10 GB budget.

The 7-day rule does eventually clear them — the trap is that **the accumulation rate can
outrun the eviction window**. Twenty-one PRs merged in ten days accumulate faster than a
seven-day clock removes. Nothing is broken; the ceiling simply arrives first, and then
LRU starts evicting the install caches that are the expensive ones to rebuild.

The fix is a workflow, not discipline:

```yaml
name: Cache cleanup
on:
  pull_request:
    types: [closed]
permissions:
  actions: write
  contents: read
jobs:
  cleanup:
    runs-on: ubuntu-latest
    steps:
      - env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          REPO: ${{ github.repository }}
          REF: refs/pull/${{ github.event.pull_request.number }}/merge
        run: |
          set -euo pipefail
          ids=$(gh api --paginate "repos/$REPO/actions/caches?per_page=100" \
                  --jq ".actions_caches[] | select(.ref == \"$REF\") | .id")
          for id in $ids; do
            gh api -X DELETE "repos/$REPO/actions/caches/$id" --silent
          done
```

**Delete by id, filtered on the ref — never by key.** `gh cache delete` matches on the
_key_, and keys repeat across refs by design (every PR has a
`Linux-bun-<lockfile hash>`). Deleting by key takes another branch's cache with it.

Caches created by tag-triggered release workflows are not covered by a
`pull_request: closed` trigger. Measure before adding a second workflow for them: at
~30 MB a release they are usually decades away from mattering, and the 7-day rule handles
them.

## Rule 2: an immutable entry means a per-SHA key is a leak

The canonical turbo/nx recipe keys on the commit:

```yaml
key: ${{ runner.os }}-turbo-${{ github.sha }}
restore-keys: ${{ runner.os }}-turbo-
```

That is correct for freshness and wrong for volume. Because entries are immutable, a new
key means a **new entry**, so every push to an open PR mints one — multiplied by every job
that caches. Nine jobs and a PR with ten pushes is ninety entries for one PR.

Key on the inputs instead, the same shape as the install cache:

```yaml
key: ${{ runner.os }}-turbo-${{ hashFiles('bun.lock', 'turbo.json') }}
restore-keys: ${{ runner.os }}-turbo-
```

The build tool re-validates each task's own input hash, so an entry that predates the
current code is partially reused, never wrongly reused. You trade a little freshness for
a bounded number of entries.

## Rule 3: what LRU evicts first is what you least want to lose

Eviction does not know that a 340 MB install cache saves 40 s and a 2 MB task cache saves
3 s. Keeping the total well under the ceiling is what protects the expensive entries. In
practice that means: clean up on PR close, and do not mint entries per push.

## Checking a repository

```bash
gh api repos/<owner>/<repo>/actions/cache/usage \
  --jq '"\(.active_caches_count) entries, \(.active_caches_size_in_bytes/1048576|floor) MB"'

# what is actually holding the space, by ref
gh api --paginate "repos/<owner>/<repo>/actions/caches?per_page=100" \
  --jq '.actions_caches[] | [.ref, .key[0:40], (.size_in_bytes/1048576|floor)] | @tsv'
```

The `usage` endpoint is eventually consistent — after a bulk delete it lags the real list
by several minutes. Trust the enumeration.

The limit is **per repository**, not per account, so a busy repo cannot starve a quiet
one. Actions _minutes_ are the opposite: they are account-wide, which is how one
repository's spending blocks another's releases.

## Checklist

- [ ] A `pull_request: closed` workflow deletes the PR's caches, by id filtered on ref.
- [ ] No cache key contains `github.sha`.
- [ ] Cache usage checked after a busy week; the ceiling is 10 GB.
- [ ] Artifact retention reviewed separately — _that_ one is billed.

## Related

- [ci-runner-cost.md](./ci-runner-cost.md) — the billed axis: runner multipliers and the
  trigger ladder.
- [pr-checks.md](./pr-checks.md) — required checks, and the gate pattern for a job that
  does not always run.
