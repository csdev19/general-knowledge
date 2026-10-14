# Pinned actions and Dependabot

_Why a workflow that holds a secret pins every action to a full commit SHA, why that pin needs an
automatic bump to be safe to keep, and what the bump does not check for you._

## The problem a pin solves

`uses: owner/action@v4` does not name code, it names a **tag**, and a tag is a pointer its owner
(or anyone who takes over the owner's account) can move. Move `v4` to a commit that prints the
job's environment and every repository using `@v4` runs it on its next trigger, with whatever
secrets that job holds. Nothing in your repository changes, so nothing in your review sees it.

A full 40-character commit SHA cannot be moved: the code it names is the code that ran last
week. That is the whole benefit, and it is why this belongs on every job that holds a deploy
token, a signing key or a release PAT (see the "mutated action tag" row in
[public-repo-production-protection.md](./public-repo-production-protection.md)).

Pin to the SHA of the **latest tag in the major you already use**, and keep the tag as a
trailing comment so a human can read it:

```yaml
uses: actions/checkout@<40-char-sha> # v5.1.0
```

Resolve it from the tag, not from memory, and check it is a commit (an annotated tag has its own
SHA that points at the commit, and `uses:` needs the commit's):

```bash
gh api repos/<owner>/<action>/git/ref/tags/<tag> --jq '.object.type, .object.sha'
# type "tag" -> dereference once more: gh api repos/<owner>/<action>/git/tags/<sha> --jq .object.sha
gh api repos/<owner>/<action>/commits/<sha> --jq .sha   # must succeed
```

## Why a pin alone goes bad

A pin freezes you, including against fixes. Left alone it rots: the action's own security patches
never arrive, the runtime it targets gets deprecated, and nobody remembers how to re-resolve a
SHA, so the pins stay for years. The usual outcome of "pin everything, update never" is a
repository that is safer on day one and worse on day three hundred.

**Dependabot is what makes pinning sustainable.** For `github-actions` it opens one PR per
action that replaces the SHA and the version comment together, so an update is a reviewable
diff with the new tag's release notes attached, not a silent tag move. The same applies to the
lockfile (`bun`, `npm`, ...): advisories against a locked version surface as PRs instead of
waiting for someone to run an audit.

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: github-actions
    directory: /
    schedule: { interval: weekly }
    cooldown: { default-days: 7 }
  - package-ecosystem: bun        # or npm; follows the lockfile
    directory: /
    schedule: { interval: weekly }
    cooldown: { default-days: 7 }
```

`cooldown` makes Dependabot ignore a version for N days after it is published. A hijacked or
broken release is often caught or yanked within days, so the delay turns "everyone takes it instantly"
into "the ecosystem found it first".

## What it does NOT do

- **It does not validate the update.** A Dependabot PR is a diff, not a test result. If your CI
  runs `verify` only on release PRs (see [release-gated-verification.md](./release-gated-verification.md)),
  these PRs show their checks as _skipped_ and `mergeable` only means "no conflicts".
- **It does not tell a patch from a rewrite.** An action jumping `v3 -> v4` can rename inputs
  and the first run that finds out is the deploy. Majors on the deploy path deserve a manual
  read of the changelog, or a dispatch of the validation workflow on the PR branch.
- **It does not coordinate lockfile PRs.** Several PRs touching the same lockfile conflict after
  the first merge; `@dependabot rebase` (or grouping) is the cure.
- **It does not replace the other half.** Pinning protects the actions; least-privilege
  `permissions:`, no cache in the job that holds the secret and a ref guard on manual runs
  protect what those actions can reach.

## Expect a burst on day one

Turning it on in a repository with old pins opens a PR for **every** stale dependency at once.
That is the backlog becoming visible, not a malfunction. Decide the policy before enabling it:
group by ecosystem, or `ignore` majors until you choose to take them, and merge deploy-path
actions on a day you can watch the next deploy.

## Check it

After the file lands, GitHub's Dependabot tab shows a config error if the file is invalid; no
error and the first PRs within minutes means it is read.

## Measured example

A one-page site on a Cloudflare Worker (the owner's `my-pets-web`) pinned its 5 actions and added
the file above in one change. Within a minute of the merge Dependabot opened 8 PRs: 4 actions
(each a major jump, including the one that runs the deploy) and 4 lockfile PRs. All 8 showed
`mergeable: CLEAN` with CI skipped, because validation ran only on the release PR.

## Related

- [public-repo-production-protection.md](./public-repo-production-protection.md) — the attack
  this closes ("a mutated action tag") and the Environment that guards the secrets.
- [release-gated-verification.md](./release-gated-verification.md) — why bump PRs often have no
  checks, and the `verify` job a release sits behind.
- [actions-cache-lifecycle.md](./actions-cache-lifecycle.md) — the cache a secret-holding job
  should not restore.
- GitHub: [Dependabot options reference](https://docs.github.com/en/code-security/dependabot/working-with-dependabot/dependabot-options-reference) ·
  [Security hardening for GitHub Actions](https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions#using-third-party-actions)
