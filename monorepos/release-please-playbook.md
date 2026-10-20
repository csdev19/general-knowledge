# Adopting release-please — implementation playbook

_Copy-paste steps to put release-please into any monorepo: pick the version topology, drop in four
files, wire the token, retire the `production` branch. The rationale lives in
[release-automation.md](./release-automation.md); this doc is the procedure._

**Read [Step 1](#step-1--decide-the-version-topology) even if you are in a hurry.** Everything else
is mechanical; the topology is the only decision, and getting it wrong ships broken pairs to
production.

---

## What you end up with

- `main` is integration. **Production is a tag.** No `production` / `hotfix` long-lived branch.
- A rolling **release PR** maintained by the bot: it accumulates the changelog and recomputes the
  next version on every merge. Merging that PR _is_ the act of releasing.
- The version that ships is a committed fact in the repo, so the tag and the built artifact can
  never disagree.
- Deploy workflows trigger on tags, so every deploy is re-runnable and rollback is "re-run the run
  of the older tag".

---

## Step 1 — Decide the version topology

### The closure rule

> A deployable's **closure** is the app itself plus every workspace package it imports,
> transitively. **Two deployables whose closures intersect must share one version line.** Only
> deployables with disjoint closures may be versioned independently.

The reason is not aesthetic. If `apps/web` and `apps/api` both import `packages/domain` and they
version independently, then a commit that changes `domain` routes — by path — to at most one of
them. That one releases and deploys; the other keeps running its last tag. **Production now runs
two different builds of the same shared code at the same time.** The bugs this produces are the
worst kind: the code is identical in `main`, and only the deployed pair disagrees.

The same commit-routing mechanic is what makes the naive policy ("bundle the shared-package change
into the PR of the app that consumes it") insufficient the moment a package has **more than one
deployed consumer**.

### The procedure

1. **List the deployables** — the things that have a deploy pipeline. Not every app: a docs site
   nobody deploys and a mobile app with no build pipeline are not deployables yet.
2. **Compute each closure** — cheapest version:

   ```bash
   for app in apps/*/package.json; do
     echo "== $app"; grep -o '"@<scope>/[a-z-]*"' "$app" | sort -u
   done
   ```

   (Go transitive if your packages depend on each other.)

3. **Intersect them** and pick a topology:

| Situation                                                     | Topology                                                    |
| ------------------------------------------------------------- | ----------------------------------------------------------- |
| Closures **intersect** (typical web + api sharing `domain`)   | **A — single version line.** One component, one tag, several deploy workflows on that tag. |
| Closures **disjoint** (genuinely separate products in one repo) | **B — independent version lines**, one component per deployable. |
| Consumer you **cannot force to update** (mobile, desktop, published SDK) | **Always its own line**, plus a compatibility contract — see below. |

### The one exception: a compatibility contract

Independent lines over intersecting closures are legitimate **only** if you accept an explicit
contract, in writing, in the workflow header:

> Changes to the shared boundary must be backward compatible with whatever version of the other
> deployable is currently in production. Add the optional field in one release; start requiring it
> in the next. Never rename or delete in the same release that stops using it. Two releases, not
> one.

Take this deal when the release cadences genuinely differ — a native app behind a 40-minute build
and a store review cannot be co-versioned with a 90-second Worker deploy, and its users update
whenever they feel like it, so you owe them backward compatibility anyway. Do **not** take it just
to get prettier per-app version numbers.

### Do not version packages you do not publish

If every workspace package is consumed through `workspace:*` and none is published to a registry,
its version number is decoration: nobody resolves it. Versioning them individually buys changelog
noise and a plugin dependency (`node-workspace` / `linked-versions`) whose behavior you then have to
verify. Under Topology A the shared packages are already covered — their commits count toward the
single line. Reach for per-package versions only when something is actually published, or when
disjoint deployables each need their own dependency bookkeeping.

---

## Step 2 — The files

Four files, all at the repo root except the workflows. Names are fixed by convention (the action
looks for them by default, and both are passed explicitly below anyway).

### Topology A — single version line

`release-please-config.json`:

```json
{
  "$schema": "https://raw.githubusercontent.com/googleapis/release-please/main/schemas/config.json",
  "release-type": "node",
  "include-component-in-tag": false,
  "bootstrap-sha": "<full sha of current main HEAD>",
  "packages": {
    ".": {
      "changelog-path": "CHANGELOG.md",
      "exclude-paths": ["apps/mobile", "apps/documentation"],
      "changelog-sections": [
        { "type": "feat", "section": "Features" },
        { "type": "fix", "section": "Bug Fixes" },
        { "type": "perf", "section": "Performance Improvements" },
        { "type": "revert", "section": "Reverts" },
        { "type": "refactor", "section": "Code Refactoring" },
        { "type": "chore", "section": "Miscellaneous Chores", "hidden": true },
        { "type": "docs", "section": "Documentation", "hidden": true },
        { "type": "style", "section": "Styles", "hidden": true },
        { "type": "test", "section": "Tests", "hidden": true },
        { "type": "build", "section": "Build System", "hidden": true },
        { "type": "ci", "section": "Continuous Integration", "hidden": true }
      ]
    }
  }
}
```

`.release-please-manifest.json`:

```json
{ ".": "1.0.0" }
```

Tags come out as `v1.1.0` — `include-component-in-tag: false` is what drops the component prefix;
manifest mode adds one by default, which for a root package would produce `<repo-name>-v1.1.0`.
`exclude-paths` keeps commits under apps with no deploy pipeline from cutting a release; drop the key
when there are none. Add `"bump-minor-pre-major": true` while pre-1.0.

### Topology B — independent version lines

`release-please-config.json`:

```json
{
  "$schema": "https://raw.githubusercontent.com/googleapis/release-please/main/schemas/config.json",
  "release-type": "node",
  "separate-pull-requests": true,
  "include-component-in-tag": true,
  "bootstrap-sha": "<full sha of current main HEAD>",
  "packages": {
    "apps/web": { "component": "web", "changelog-path": "CHANGELOG.md" },
    "apps/native": { "component": "native", "changelog-path": "CHANGELOG.md" }
  }
}
```

`.release-please-manifest.json`:

```json
{ "apps/web": "1.10.0", "apps/native": "1.2.0" }
```

Tags come out as `web-v1.10.0` / `native-v1.2.0`, one independently mergeable release PR each.

### The cutter — `.github/workflows/release-please.yml`

Identical under both topologies. It cuts releases and **only** cuts releases; deploys live in their
own tag-triggered workflows so one failing deploy can never leave another half-shipped.

```yaml
name: Release Please 🏷️

on:
  push:
    branches: [main]

permissions:
  contents: write
  pull-requests: write

concurrency:
  group: release-${{ github.ref }}
  # Never cancel a half-finished release cut — queue instead.
  cancel-in-progress: false

jobs:
  release-please:
    name: Release Please
    runs-on: ubuntu-latest
    # RELEASE_PLEASE_TOKEN lives in the `production` environment, where the repo
    # already keeps its secrets. A job only sees an environment's secrets when it
    # declares that environment; `production` has no protection rules, so this
    # does not gate the run.
    environment: production
    steps:
      - uses: googleapis/release-please-action@v4
        with:
          token: ${{ secrets.RELEASE_PLEASE_TOKEN }}
          config-file: release-please-config.json
          manifest-file: .release-please-manifest.json
```

### The deployers — one `release-<thing>.yml` per deployable

Take the steps you already have in `deploy-production.yml` and split them, one workflow per
deployable. Only the trigger and the concurrency policy are new:

```yaml
name: Release Web 🚀

# Deploys apps/web. Trigger: the tag cut by release-please.yml.
# The tag ref is what gets checked out, so the deploy is exactly the released commit.
#
# Re-deploy / rollback: re-run THIS workflow's run for the wanted tag from the
# Actions UI — that re-checks out the tag. `workflow_dispatch` is different: it
# checks out whatever ref you pick, defaulting to the default branch, so it can
# ship unreleased commits. Escape hatch only.

on:
  push:
    tags: ["v*.*.*"] # Topology B: ["web-v*.*.*"]
  workflow_dispatch:

concurrency:
  group: ${{ github.workflow }}
  # Stateless deploy: newest wins. Use `false` for anything that runs migrations —
  # cancelling between deploy and migrate leaves a schema half-applied.
  cancel-in-progress: true

jobs:
  release-web:
    runs-on: ubuntu-latest
    environment: production
    steps: # …exactly the steps from the old deploy-production job…
```

Under Topology A several workflows listen to the **same** tag. That is deliberate: one release, one
commit, but independent runs — separate logs, separate concurrency, and you re-run only the one
that failed.

---

## Step 3 — The token (nothing works without it)

A tag pushed by the default `GITHUB_TOKEN` **does not trigger** `on: push: tags` workflows — GitHub
suppresses events raised by that token to prevent recursion. With the default token the release PR
opens and merges, the tag appears, and **nothing deploys**.

Manual, one time, by the repo owner:

1. Create a **fine-grained PAT** with `Contents: read/write` + `Pull requests: read/write` on the
   repo (classic PAT with `repo` also works).
2. Add it as **`RELEASE_PLEASE_TOKEN`** in the `production` environment (or wherever the job's
   `environment:` points).

On some private org repos the default token also makes release-please's GraphQL call fail with a
500 — a second, independent reason to use the PAT.

---

## Step 4 — Seed the versions

- Put each component's **current** version in `.release-please-manifest.json`, and make sure the
  matching `package.json` carries the same `version` field. A `package.json` with **no** `version`
  key breaks the `node` releaser — add it before the first run.
- If tags already exist, seed to the **highest existing tag** so the bot always moves forward
  instead of colliding. Do not delete old tags; the next release supersedes them.
- Pre-1.0 repos: add `"bump-minor-pre-major": true` so a `feat!` bumps minor instead of declaring
  `1.0.0` by accident. Declare `1.0.0` deliberately later with a `Release-As: 1.0.0` commit footer.
- `bootstrap-sha` = the full SHA of `main` at adoption. Without it the bot walks the entire history
  and drags years of pre-convention commits into the first changelog.

---

## Step 5 — Retire the `production` branch

1. Delete `deploy-production.yml` (that also removes any `hotfix` trigger riding on it).
2. Leave the remote branch in place at first — it is now inert, and the change stays revertible.
   Delete it once a tag-driven deploy has succeeded.
3. Fix the mental model everywhere it is written down: **`main` is the current state of the code;
   production is a tag.** Promotion is cutting a tag, not merging a branch.
4. Between deleting the old workflow and the first successful tag deploy **there is no path to
   production**. Do it deliberately, not on a Friday.

---

## Step 6 — Keep release PRs out of the heavy CI, but verify them

A release PR touches `package.json` and `CHANGELOG.md`, so it matches most path filters and would
otherwise run the full build/test matrix on a mechanical version bump. Skip the matrix:

```yaml
jobs:
  validate:
    if: ${{ !startsWith(github.head_ref, 'release-please--') }}
```

**Do not skip `verify` on it.** The release PR is not mechanical from your checks' point of view:
it is the only PR whose files a bot wrote, and bot commits never pass the pre-push hook. The first
check that sees `CHANGELOG.md`, the manifest and the bumped `package.json` is otherwise the
release workflow's `verify` — after release-please has already created the tag and the GitHub
Release, so a failure burns the version. Run the cheap Linux `verify` on the release PR when it is
marked ready, and only there; the shape that costs nothing on ordinary PRs is in
[release-gated-verification.md](./release-gated-verification.md#a-required-check-with-candidate-only-verification).

**Keep the generated files out of the formatter check.** release-please writes changelogs in its
own style (`*` bullets, double blank lines) and rewrites them on every release, so a formatter
check (`prettier --check`, `oxfmt --check`, `biome format`) rejects them, and formatting them by
hand is undone by the next release. Ignore them in the formatter's config:

```jsonc
// .oxfmtrc.json / .prettierignore equivalent
{ "ignorePatterns": ["**/CHANGELOG.md"] }
```

---

## Verification checklist

1. Land the config + manifest + workflows on `main`.
2. **The release PR appears** with a sane version and a changelog containing only the commits since
   `bootstrap-sha`. Nothing has shipped yet — opening a PR is free.
3. **`verify` passes on the release PR branch** (`gh pr checkout <n> && bun run verify`, or the
   candidate job of Step 6). The first release is the first time the generated files meet your
   formatter and linters.
4. Merge it. **The tag exists** and it points at the merge commit that carries the bumped
   `package.json`.
5. **The deploy workflows actually started.** If the tag exists but no workflow ran, the token is
   wrong (Step 3) — that is the single most common failure.
6. The deployed build reports the released version.
7. Re-run the deploy run from the Actions UI once, to prove rollback works before you need it.

---

## Gotchas

| Symptom                                                        | Cause                                                                                          |
| -------------------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| Tag created, no deploy ran                                     | Default `GITHUB_TOKEN` — use the PAT (Step 3).                                                 |
| No release PR at all                                           | The batch is only `chore`/`docs`/`refactor`/`ci` — nothing releasable. Land a `feat`/`fix`.     |
| No release PR after a `packages/*`-only change                 | Topology B: shared-package commits belong to no component by path. This is the closure rule biting — see Step 1. |
| First release PR contains hundreds of commits                  | Missing `bootstrap-sha`.                                                                       |
| `Cannot read properties of undefined (reading 'version')`      | The released `package.json` has no `version` field.                                            |
| An app you never deploy keeps cutting releases                 | Add it to `exclude-paths`.                                                                     |
| Want a specific next version (e.g. declare `1.0.0`)            | `Release-As: 1.0.0` footer in a commit body.                                                   |
| Release workflow's `verify` fails on `CHANGELOG.md` after the tag exists | The formatter checks release-please's generated changelogs, and nothing verified the release PR. Ignore `**/CHANGELOG.md` in the formatter and verify the release PR (Step 6). Ship the fix as a patch release; the failed version stays burnt. |
| A group release PR merged but never tagged; every later run logs `untagged, merged release PRs outstanding - aborting` | Seen with `linked-versions` + `separate-pull-requests: true` (release-please 17.3.0). The plugin titles the group PR `chore${scope}: release <group> libraries`, but a merged title is parsed only with the component pattern or the top-level `group-pull-request-title-pattern`, which defaults to `chore: release ${branch}`. Neither matches, so building the release logs `Bad pull request title`, and **every later run aborts, for every component**, until that PR is tagged. Fix: set `"group-pull-request-title-pattern": "chore${scope}: release${component} libraries"` at the top level of the config (versions still come from the PR body); the next run on `main` then tags the stuck PR. Set it before the first grouped release. |
| Considering `node-workspace` / `linked-versions` on **Bun**    | Bun's root `workspaces` is an **object** (`{ packages: [], catalog: {} }`), not the array these plugins expect. Verify with a dry run before committing to it; Topology A avoids the question entirely. |

Commit type → bump table: see
[release-automation.md § Commit → bump reference](./release-automation.md#commit--bump-reference).
