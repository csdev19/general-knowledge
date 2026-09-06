# Git hooks with lefthook — decision and playbook

_Why the hub standardizes on [lefthook](https://lefthook.dev) for git hooks, the hook contract every
repo follows (commit fixes, push verifies, commit-msg guards the release pipeline, CI backstops), the
files to drop in, and the migration procedure from husky + lint-staged. Verified against lefthook
2.1.12 (2026-08-28) and Bun 1.3.4._

**The library swap is the least important part of this doc.** The gain is the hook contract in
[Step 1](#step-1--the-hook-contract) and the fact that it lives in one declarative file that can be
copied between repos without drift. If you only replace `husky` with `lefthook` and keep the old
hooks' behaviour, you got a smaller `node_modules` and nothing else.

---

## What you end up with

- **One file, `lefthook.yml`**, declares every hook. No `.husky/` shell scripts, no `lint-staged`
  block in `package.json`, no `prepare: husky` boilerplate. `lefthook dump` prints the resolved
  config; `lefthook validate` checks it.
- **A single Go binary**, installed through the `lefthook` npm package (platform binary as an
  optional dependency). Bun trusts its postinstall by default, so `bun install` installs the hooks.
- **The hook contract**: `pre-commit` fixes what is being committed (format + lint on staged files,
  fixes re-staged), `pre-push` verifies what is being shared with read-only checks identical to CI,
  `commit-msg` rejects messages release-please cannot read.
- **Per-developer overrides** in a gitignored `lefthook-local.yml`, `LEFTHOOK=0` to bypass once,
  `skip: [merge, rebase]` so hooks stay out of the way where they only add noise.
- **CI can run the same hook** (`lefthook run pre-push --all-files`) instead of re-declaring the
  commands, so "green locally" and "green in CI" mean the same thing.

---

## Why lefthook, and what it gives beyond a library swap

### What husky + lint-staged actually cost

This is what the audit of thirteen repos sharing the same husky setup found in September 2026:

1. **The configuration is split across three places** that must agree by hand: a `lint-staged`
   block in `package.json` (which globs run which commands on commit), a `.husky/pre-commit` file
   (`lint-staged`), and a `.husky/pre-push` shell script (30 lines of `git ls-files | grep | xargs`,
   exit-code checks and emoji echo). Nothing validates them together.
2. **The shell boilerplate drifts and rots.** Every one of the thirteen repos carried a
   `pre-push` that still sourced `_/husky.sh` — the husky 4 idiom. husky 9 prints
   `husky - DEPRECATED ... They WILL FAIL in v10.0.0` on every push, and nobody noticed because it
   is one warning among the tool output. The template repo had the same lines, so every new
   project inherited the time bomb.
3. **The pre-push hook was doing the wrong job.** It ran `oxfmt --write` over every tracked file
   and then `git add -u`. Staging changes during a push does not change the commits being
   pushed: the unformatted commits go out, the fixes stay behind as a dirty index, and CI's
   `oxfmt --check .` is what really decides. The hook produced surprise, not safety.
4. **No per-developer escape hatch.** husky sets `core.hooksPath` to `.husky/_` and the scripts
   are committed, so "skip this one hook on my machine" means editing a tracked file or
   `--no-verify`, which skips everything.
5. **`lint-staged` exists only to filter files and run commands** — a Node process with its own
   dependency tree started on every commit to do what a hooks manager should do natively.

### What lefthook gives

| Need | lefthook | husky + lint-staged |
| --- | --- | --- |
| Declare hooks | one `lefthook.yml` | `.husky/*` scripts + `lint-staged` block |
| Run on staged files only | `{staged_files}` + `glob` | lint-staged |
| Run on files about to be pushed | `{push_files}` | hand-rolled `git` plumbing |
| Re-stage formatter fixes | `stage_fixed: true` | lint-staged (implicit) |
| Parallel jobs | `parallel: true` | no |
| Skip during merge / rebase / on a branch | `skip: [merge, rebase, {ref: main}]` | shell `if` |
| Per-developer override | `lefthook-local.yml` (gitignored) | none |
| Bypass once | `LEFTHOOK=0 git commit` | `HUSKY=0 git commit` |
| Reuse the hook in CI | `lefthook run pre-push --all-files` | re-declare the commands |
| Monorepo sub-package with its own tooling | `root: apps/mobile/` | `cd` in shell |
| Runtime | one Go binary | Node + lint-staged's dependency tree |
| Validate the config | `lefthook validate`, `lefthook dump` | run a commit and see |

### What it does **not** give in this stack — be honest about speed

The internet sells lefthook on speed. With `oxlint` and `oxfmt` (Rust) the linters are already the
fast part; the measurements on a ~850-file monorepo were:

| Step | Wall time |
| --- | --- |
| `oxlint` on the whole repo | 0.1 s |
| `lint-staged` startup with nothing staged | 0.16 s |
| `oxfmt --check .` on the whole repo | 1.4 s |

Nothing here is worth a migration. **Do not justify this decision on speed.** Justify it on the
contract, the single config file, the removed shell boilerplate, and the escape hatches.

### Alternatives rejected

- **Keep husky, delete the two deprecated lines.** Cheapest fix for the v10 time bomb, but keeps
  config in three places, the hand-rolled `pre-push`, and no local override. It would also leave
  the hub with two contradicting recommendations (husky here, hand-rolled `.githooks` in the CI
  strategy doc).
- **Hand-rolled `.githooks/` + `core.hooksPath`.** Zero dependencies, fully transparent, but every
  repo re-implements file filtering, staging of fixes and skip logic in shell — the drift problem
  again, and `core.hooksPath` must be set by hand on every clone.
- **simple-git-hooks.** Minimal and honest, but it only maps hook → command in `package.json`; it
  has no file filtering, so lint-staged stays, and the split config stays.
- **hk** (jdx, Rust, Pkl config). Interesting design (file-level read/write locks between parallel
  linters, better partial-staging story) but young, and Pkl is one more language for a config that
  should be readable by everyone. Reopen if lefthook's parallel writer race ever bites.
- **pre-commit** (Python). Broadest hook ecosystem, wrong runtime for a Bun/TypeScript stack.

### The condition that reopens this decision

- lefthook stops being maintained (it is by Evil Martians, 2.x released 2025-10, patch releases
  monthly as of 2026-08), **or**
- a hook needs two writers on the same files in parallel (lefthook has no file locking; `hk`
  does), **or**
- Bun removes `lefthook` from its default trusted-dependencies list *and* pnpm-style opt-in
  becomes a support burden across repos.

---

## Step 1 — The hook contract

This is the decision each hook implements. Copy it into the repo's docs; the YAML is only its
encoding.

| Hook | Job | Rule |
| --- | --- | --- |
| `pre-commit` | **Fix what you are committing.** Format + lint only the staged files; re-stage the fixes (`stage_fixed`). | Never touches files that are not being committed. Sequential when a job writes (format) and another reads (lint) the same files — see gotchas. |
| `commit-msg` | **Guard the release pipeline.** Reject a first line that is not a Conventional Commit. | Only where release-please (or anything else) parses commit messages. Skipped on merge/rebase; `fixup!`/`squash!`/`Merge`/`Revert` pass. No commitlint dependency: one `grep`. |
| `pre-push` | **Verify what you are sharing.** Read-only: `oxlint` + `oxfmt --check`, in parallel. | Must be the same commands CI runs, so a green push predicts a green PR. **Never writes, never stages.** |
| CI | **Backstop.** Same commands as `pre-push`, plus types, build and tests. | Hooks are advice (`--no-verify` exists); CI is the guarantee. See [ci-cd-pipeline-strategy](../conventions/ci-cd-pipeline-strategy.md). |

Why `pre-push` does not type-check or build: those are the slow steps, they are cached in CI, and
a hook that takes ten seconds gets skipped. Add `bun run check-types` to `pre-push` only when CI
type failures become the common reason PRs go red; that is the signal, not a preference.

---

## Step 2 — The files

### `lefthook.yml` — oxc stack (`oxlint` + `oxfmt`)

```yaml
# Git hooks. Docs: https://lefthook.dev — contract: <link to your docs or this playbook>.
# commit = fix what you commit · push = verify what you share (read-only, same as CI)
# commit-msg = reject what release-please cannot parse. Bypass once: LEFTHOOK=0 git <cmd>.
min_version: 2.0.0

pre-commit:
  jobs:
    - name: format
      glob: "*.{js,jsx,cjs,mjs,ts,tsx,cts,mts,json,jsonc,css,md,mdx}"
      run: bunx oxfmt --write {staged_files}
      stage_fixed: true
    - name: lint
      glob: "*.{js,jsx,cjs,mjs,ts,tsx,cts,mts}"
      run: bunx oxlint {staged_files}

commit-msg:
  skip:
    - merge
    - rebase
  jobs:
    - name: conventional-commit
      run: |
        first="$(head -1 {1})"
        echo "$first" | grep -qE '^(fixup!|squash!|Merge |Revert )' && exit 0
        echo "$first" | grep -qE '^(feat|fix|docs|style|refactor|perf|test|build|ci|chore|revert)(\([a-zA-Z0-9._/-]+\))?!?: .+' && exit 0
        echo "Commit message must follow Conventional Commits, e.g. 'feat(web): add recordings page'."
        echo "release-please derives versions and changelogs from these prefixes."
        exit 1

pre-push:
  parallel: true
  jobs:
    - name: lint
      run: bunx oxlint
    - name: format-check
      run: bunx oxfmt --check .
```

`lefthook validate` passes on this file, `lefthook dump` shows it resolved as written, and the
`commit-msg` regex was checked against `feat(web): add page`, `fix: typo`, `chore!: drop`,
`Merge branch 'x'`, `fixup! feat: x` (accepted) and `feta: typo`, `update stuff` (rejected).

### `lefthook.yml` — Biome stack

Same contract, different tools. Biome formats and lints in one pass, so `pre-commit` is one job:

```yaml
min_version: 2.0.0

pre-commit:
  jobs:
    - name: biome
      glob: "*.{js,jsx,cjs,mjs,ts,tsx,cts,mts,json,jsonc,css}"
      run: bunx biome check --write {staged_files}
      stage_fixed: true

pre-push:
  jobs:
    - name: biome
      run: bunx biome ci .
```

Add the `commit-msg` block from the oxc variant unchanged when the repo uses release-please.

### `package.json`

```jsonc
{
  "scripts": {
    "prepare": "lefthook install"
  },
  "devDependencies": {
    "lefthook": "^2.1.12"
  }
}
```

- `prepare` is deliberate even though the npm package installs hooks in `postinstall`: it makes
  the install explicit, it covers pnpm repos (which block postinstall unless allow-listed), and it
  **fails loudly** when a stale `core.hooksPath` from husky is still set — the exact fix is in the
  error. Do not use `lefthook install -f` here: `-f` "succeeds" by installing into the stale path.
- Remove `husky`, `lint-staged`, the `"lint-staged": {}` block and `prepare: "husky"`.

### `.gitignore`

```
lefthook-local.yml
```

### Optional: `lefthook-local.yml` (per developer, never committed)

```yaml
pre-push:
  jobs:
    - name: format-check
      skip: true   # e.g. while bisecting an old branch
```

Named jobs merge by name, so a local file overrides one job without repeating the others.

---

## Step 3 — Migrating a repo from husky + lint-staged

Run from the repo root, on a branch.

```sh
# 1. Swap the packages
bun remove husky lint-staged
bun add -d lefthook

# 2. Drop the old files and config
git rm -r .husky
#    package.json: delete the "lint-staged" block, set "prepare": "lefthook install"

# 3. Add the new config
#    lefthook.yml from Step 2 · append lefthook-local.yml to .gitignore

# 4. Reset the clone-local pointer husky left behind (once per existing clone)
git config --unset-all --local core.hooksPath

# 5. Install and check
bun install                 # runs prepare → lefthook install
bunx lefthook validate
bunx lefthook dump
ls .git/hooks | grep -vE '\.sample$'   # pre-commit  commit-msg  pre-push
```

Step 4 is the one people miss. `git config core.hooksPath` printing `.husky/_` means git is still
looking at a directory you just deleted: **no hook runs, and nothing tells you.** Fresh clones do
not have this problem. `bun install` without step 4 fails at `prepare` with lefthook's message
naming the three ways out; pick the `--unset-all --local` one.

Then update the docs that mentioned husky (README setup section, stack briefing, any "add X to
husky" note) and open the PR with a body that tells existing clones to run step 4.

---

## Step 4 — Rolling out across many repos

1. **The template repo first.** Every new project is a copy of it; migrating it stops the bleed.
2. **The repos with an active release pipeline next** (they gain the `commit-msg` guard).
3. **Idle repos on their next touch**, not in a batch: a hooks PR on a repo nobody is committing to
   is noise.

Each migration is one small PR: `chore(hooks): migrate husky + lint-staged to lefthook`. Copy
`lefthook.yml` verbatim from the template; do not "improve" it per repo — sameness is the point.

---

## Verification checklist

- [ ] `git config core.hooksPath` prints nothing.
- [ ] `ls .git/hooks` lists `pre-commit`, `commit-msg`, `pre-push` (non-`.sample`).
- [ ] `bunx lefthook validate` → `All good`.
- [ ] Stage a badly formatted `.ts` file, commit: the commit contains the formatted version
      (`git show --stat HEAD` lists the file, `bunx oxfmt --check <file>` is clean).
- [ ] Stage a file with a lint error, commit: rejected with the `oxlint` output.
- [ ] `git commit -m "update stuff"` on a staged change: rejected by `commit-msg`;
      `git commit -m "chore: update stuff"` passes.
- [ ] `bunx lefthook run pre-push` runs `lint` and `format-check` in parallel and exits 0 on a
      clean tree; introduce a formatting error in a committed file and it exits 1 **without
      modifying the file**.
- [ ] `LEFTHOOK=0 git commit -m "update stuff" --allow-empty` goes through (bypass works).
- [ ] CI is green on the migration PR with the same lint / format commands it already ran.

---

## Gotchas

- **`core.hooksPath` is per clone, not per repo.** husky set it locally on every machine that ever
  ran `bun install`. Deleting `.husky/` does not unset it. See Step 3, step 4.
- **The npm postinstall runs `lefthook install -f`.** With a stale `core.hooksPath` that means
  "install into `.husky/_` anyway" — hooks work, but you keep an untracked `.husky/_` forever. The
  `prepare: "lefthook install"` script (no `-f`) is what turns that into a loud error.
- **Bun runs lefthook's postinstall; pnpm does not.** `lefthook` is on Bun's default
  trusted-dependencies list (verified in `src/install/default-trusted-dependencies.txt`, Bun 1.3).
  pnpm needs `pnpm.onlyBuiltDependencies: ["lefthook"]` in `package.json` — or just rely on the
  `prepare` script, which pnpm does run.
- **`CI=true` skips the postinstall hook install** (by design; hooks are pointless in CI). The
  `prepare` script still installs them there, harmlessly. If you want CI to reuse the hook, call
  `bunx lefthook run pre-push --all-files` explicitly.
- **`node_modules/.bin` is not on `PATH` inside a hook.** husky prepended it; lefthook does not.
  Write `bunx oxlint`, not `oxlint`.
- **Do not run a writer and a reader in parallel on the same files.** `oxfmt --write` and `oxlint`
  on the same staged set can race; keep `pre-commit` sequential (format, then lint). `parallel:
  true` is for `pre-push`, where every job is read-only.
- **Partially staged files.** lefthook stashes the unstaged hunks of tracked files before
  `pre-commit` and restores them after, like lint-staged. It does **not** stash untracked files,
  which only matters for jobs that run on `{all_files}` — the contract never does that on commit.
- **lefthook 2.0 (2025-10) broke older blog posts.** `exclude` is glob-only (no regex),
  `skip_output` became `output`, some CLI flags were renamed. Trust `lefthook run -h` over a 2024
  article. `commands:` still works; `jobs:` (1.10+) is the list form used here because named jobs
  merge cleanly with `lefthook-local.yml`.
- **`{1}` in `commit-msg` is the message file path**, not the message. Hence `head -1 {1}`.
- **Windows** needs a `sh` (Git Bash). Same requirement husky had; not a regression.
- **`oxfmt --check .` scans untracked directories too** (a stale `.claude/worktrees/` made it fail
  locally while CI was green). Either keep the working tree clean or exclude such paths in the
  formatter config; do not weaken the hook.

---

## See also

- [monorepo-structure.md](./monorepo-structure.md) — where hooks sit in the repo tooling.
- [ci-cd-pipeline-strategy.md](../conventions/ci-cd-pipeline-strategy.md) — the tiered gates the
  `pre-push` hook is the first of.
- [release-please-playbook.md](./release-please-playbook.md) — why the `commit-msg` guard exists.
- lefthook docs: [configuration](https://lefthook.dev/configuration/),
  [`stage_fixed`](https://lefthook.dev/configuration/stage_fixed/),
  [`skip` / `only`](https://lefthook.dev/configuration/skip/),
  [`root` for monorepos](https://lefthook.dev/configuration/root/),
  [`lefthook-local.yml`](https://lefthook.dev/examples/lefthook-local/).
