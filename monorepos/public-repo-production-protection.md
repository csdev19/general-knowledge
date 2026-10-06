# Production protection on a public repository

_GitHub Actions playbook: the production Environment is reachable only from the refs that release,
pull request checks get an Environment of their own, and the settings that keep a stranger's code
out of a job that holds a secret._

## What an attacker needs, and what they can see

Nothing in a public repository exposes a secret by being read. Workflows hold `${{ secrets.X }}`
references, not values; GitHub never shows a stored value again, even to administrators; logs
mask any string that equals a secret. What an attacker needs is to **run their own code inside
a job that holds one**. Every setting below closes one way of doing that, ordered by how often
it actually happens:

| Way in | Closed by |
| --- | --- |
| A fork's pull request edits a workflow to print the secrets | `pull_request` runs with no Environment secrets and a read-only token. Keep it that way: never `pull_request_target`, `workflow_run` or `issue_comment` for anything that checks out the PR's code. |
| Code that runs during CI exfiltrates — a `postinstall`, a build script, a mutated action tag | No install scripts ([native-dependencies](../desktop/native-dependencies.md)); pin third-party actions to a full SHA where the job holds a secret. |
| A job on an ordinary branch declares the production Environment | Deployment branch policy, below. |
| The owner's account, or a PAT with bypass rights | 2FA; the release bot's PAT scoped to contents + pull requests, no ruleset bypass. |
| A secret pasted into a commit | Secret scanning with push protection (free on public repositories). |

The third row is the one a repository that went public is most likely to have open, because
nothing fails while it is.

## The production Environment is for releases only

An Environment is the boundary around a set of secrets, and its **deployment branch policy** is
what decides which refs may cross it. Unrestricted, it means every branch: a job that declares
`environment: production` on a feature branch receives the signing keys, the Cloudflare token
and the database URL, whether it meant to or not.

Restrict it to the refs that release:

| Type | Pattern |
| --- | --- |
| branch | `main` |
| tag | `desktop-v*`, `web-v*`, `api-v*` (whatever the release tags are) |

Every workflow that still declares `production` must run on one of those refs — the release
pipelines on their tags, the release-PR bot on pushes to `main`. Anything else that declares it
fails at the Environment check, before a runner is allocated, which is the failure you want.

The GitHub form sometimes rejects a tag pattern with a wildcard; the API takes it:

```sh
gh api -X POST repos/<owner>/<repo>/environments/production/deployment-branch-policies \
  -f name='web-v*' -f type=tag
```

**Uncheck "Allow administrators to bypass configured protection rules".** On a repository with
one administrator, every protection is protection against that person's own mistakes; a rule
the administrator bypasses protects nobody. The cost is adding a branch to the list before a
one-off manual deploy from it, which takes a minute and leaves a record.

## Pull request checks get their own Environment

The way `production` ends up on every branch is innocent: a PR-time job needs one value that
lives there — a public URL the build inlines, a feature flag — and declaring the Environment is
the quickest way to read it. The job then holds every production secret to read one variable.

Give the checks an Environment of their own (`testing`, `ci`): no protection rules, no branch
restriction, holding only what the checks need. Today that may be a single variable; the day a
test needs a credential — a throwaway database, a sandbox API key — it has a home that is not
production. A repository-level variable also works for a lone public value, but it has no place
for a secret, so the question comes back.

The same reasoning puts a `preview` Environment around a preview deploy: same cloud credentials
as a release, declared under a name that does not announce "Deploying to production" on a run
that deploys nothing.

## The rest of the checklist

Each of these is a repository setting, not a workflow change:

- **Rules → Rulesets → the default branch: require a pull request before merging.** With zero
  required approvals on a single-maintainer repository — GitHub does not let an author approve
  their own PR — the rule still means every change to a workflow is seen in a PR before it runs
  with secrets. Keep "block force pushes" and "block deletion".
- **Actions → General → fork pull request workflows: require approval for all external
  contributors.** A fork's PR cannot reach secrets, but it can spend runner minutes and write
  to the Actions cache; with this, nothing runs until the maintainer has read it. (The default
  only gates first-time contributors.)
- **Actions → General → workflow permissions: read-only** for `GITHUB_TOKEN`; a workflow that
  needs more declares it in its own `permissions:` block.
- **Code security: secret scanning and push protection on; Dependabot alerts on.** Free on
  public repositories and off by default on a repository that was private.
- Forking cannot be disabled and pull requests cannot be closed as a feature. Projects that
  "closed external PRs" use a bot that closes them or **Settings → Moderation → Interaction
  limits**, which blocks non-collaborators for up to six months.

## When a repository goes public

The settings above are off on a private repository and nothing fails when it flips to public.
Audit them that day, in this order: Environment branch policies and the admin bypass; which
jobs declare `production` on `pull_request` events; the ruleset; fork approval; secret scanning.
The audit is five `gh api` reads:

```sh
R=repos/<owner>/<repo>
gh api $R/environments -q '.environments[]|[.name,(.deployment_branch_policy|tostring)]|@tsv'
gh api $R/rulesets -q '.[].id' | xargs -I{} gh api $R/rulesets/{} -q '[.rules[].type]'
gh api $R/actions/permissions/fork-pr-contributor-approval -q .approval_policy
gh api $R/actions/permissions/workflow -q .default_workflow_permissions
gh api $R -q '.security_and_analysis'
```

## Related

- [release-gated-verification.md](./release-gated-verification.md) — the `verify` job a
  publish step sits behind; this page is about who may reach the publish step's secrets.
- [pr-checks.md](./pr-checks.md) — the PR-time jobs that must not declare `production`.
- [worker-secrets-with-deploy.md](./worker-secrets-with-deploy.md) — how the secrets a release
  job holds reach the Worker.
- [../infra/infisical-secrets.md](../infra/infisical-secrets.md) — the secret manager upstream
  of GitHub's Environments.
- [../desktop/native-dependencies.md](../desktop/native-dependencies.md) — why `bun install`
  runs nothing.
- GitHub: [Deployment branches and tags](https://docs.github.com/en/actions/how-tos/deploy/configure-and-manage-deployments/manage-environments#deployment-branches-and-tags) ·
  [Keeping your GitHub Actions secure: preventing pwn requests](https://securitylab.github.com/resources/github-actions-preventing-pwn-requests/)
