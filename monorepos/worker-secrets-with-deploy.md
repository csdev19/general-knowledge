# Worker secrets ship with the deploy

_Cloudflare Workers playbook: secrets go up in the same `wrangler deploy` as the code, through
`--secrets-file`. Editing them as a separate step deploys on its own, and is refused the moment a
version has been uploaded without being deployed._

## The rule

A release job uploads code and secrets as **one version**:

```sh
wrangler deploy --secrets-file worker-secrets.json
```

It never edits secrets as a step of their own — not `wrangler secret put`, not
`wrangler secret bulk`, and not the `secrets:` input of `cloudflare/wrangler-action`, which is
the same thing wearing a convenient name. The flag exists on `deploy` and on `versions upload`
since Wrangler 4.x and is what Cloudflare documents for CI/CD.

## Why a separate secrets step is wrong, even while it works

A Worker is a stack of **versions**. A deploy creates a version and routes traffic to it; an
upload creates a version and routes nothing. Every version carries the full set of secrets.

`wrangler secret put` has to put the new value *somewhere*, so it creates a new version **from
the latest uploaded one** and deploys it. Two consequences follow, one visible and one not:

1. **It is never atomic.** With the action's `secrets:` input the order is secrets first, then
   `command`. Production serves the old code with the new secrets for the length of the build,
   then the new code. A secret whose meaning changed with the code (a renamed key, a new
   service URL) is wrong in production for that window on every release.
2. **It breaks the first time a version is uploaded without being deployed.** If the latest
   version is an upload — a `versions upload` preview, a gradual rollout, a version created
   from the dashboard — "deploy a new version from the latest one" would silently put that
   undeployed code in production. Cloudflare refuses the edit instead:

   > Secret edit failed. You attempted to modify a secret, but the latest version of your
   > Worker isn't currently deployed.

   The refusal is the guard working. The release still fails before deploying anything, and it
   fails on **every** release that follows a preview, not once.

The second consequence is why the pattern survives in repositories for months: it is sound as
long as the only thing that ever creates a version is the deploy. Adding a preview step — the
[Version URLs mechanism](../infra/deploy-environments.md#cloudflare-workers-three-mechanisms-not-one)
— is what turns it into a release outage, and nothing in the preview's own run says so.

## The job

```yaml
- name: Write Worker secrets file
  working-directory: apps/<app>
  env:
    SERVICE_URL: ${{ vars.SERVICE_URL }}
    DATABASE_URL: ${{ secrets.DATABASE_URL }}
  run: |
    umask 077
    jq -n \
      --arg SERVICE_URL "$SERVICE_URL" \
      --arg DATABASE_URL "$DATABASE_URL" \
      '$ARGS.named | with_entries(select(.value != ""))' > worker-secrets.json

- name: Deploy
  uses: cloudflare/wrangler-action@v3
  with:
    apiToken: ${{ secrets.CLOUDFLARE_API_TOKEN }}
    accountId: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
    workingDirectory: apps/<app>
    command: deploy --secrets-file worker-secrets.json

- name: Remove Worker secrets file
  if: always()
  run: rm -f apps/<app>/worker-secrets.json
```

Each line that is not obvious:

- **`jq`, not `echo`.** A value containing `"` or `\` breaks a hand-written JSON file; `jq
  --arg` escapes it. The file may also be `.env` format, which has the same quoting problem
  with no tool to solve it.
- **`with_entries(select(.value != ""))`.** `--secrets-file` is additive: a secret absent from
  the file keeps the value the Worker already has. An *empty* value is present, and blanks it.
  Dropping empties means a GitHub secret that went missing keeps production on its last good
  value instead of taking it down. Removing a secret is a deliberate `wrangler secret delete`,
  never a side effect of a release.
- **`umask 077`, `if: always()`, `.gitignore`.** The file exists on the runner for the length
  of one step, readable by the runner user only, and is removed even when the deploy fails.
  `worker-secrets.json` is in `.gitignore` so a developer replaying the step locally cannot
  commit it. None of this is what keeps the secrets private — a public repository never sees
  the file, and GitHub masks the values in logs — it is what keeps a mistake small.
- **Pin Wrangler.** `bun add -g wrangler` installs whatever is latest on the day of the
  release, so two runs of the same tag can deploy with different tools. Pin the version in
  every workflow that runs Wrangler, and keep the preview on the same version as the release
  it stands in for — see [wrangler-env-config.md](./wrangler-env-config.md).

## Related

- [wrangler-env-config.md](./wrangler-env-config.md) — where each Worker setting lives; the
  production row points here.
- [ci-cd-pipelines.md](./ci-cd-pipelines.md) — the release pipelines this step belongs to, and
  the secret / variable matrix.
- [../infra/deploy-environments.md](../infra/deploy-environments.md) — the three Cloudflare
  mechanisms; Version URLs are the one that leaves an undeployed version behind.
- [../infra/infisical-secrets.md](../infra/infisical-secrets.md) — where the values come from
  before they reach the job.
- [public-repo-production-protection.md](./public-repo-production-protection.md) — which
  branches may receive those GitHub secrets at all.
- Cloudflare: [Secrets](https://developers.cloudflare.com/workers/configuration/secrets/) ·
  [`secrets` config property](https://developers.cloudflare.com/changelog/post/2026-03-24-secrets-config-property/)
  (declare the required names in `wrangler.jsonc`; the deploy fails when one is missing).
