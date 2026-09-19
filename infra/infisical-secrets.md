# Infisical: secrets and environment playbook

> Status: **accepted default, independent of Varlock.** Last reviewed: 2026-09-18.
> Infisical stores values; existing runtime schemas validate them. Varlock is an
> optional declaration layer evaluated [separately](./varlock-evaluation.md), not
> a prerequisite for this playbook.

Use one authoritative store **per value and environment**, with consumer-scoped
delivery. A local migration does not imply production CI has migrated. Document
the actual source of each consumer until the transition is complete.

**The rule:** commit names, contracts, and non-sensitive metadata, never credential
values. Generated environment files are gitignored, owner-only, and not edited
by hand. A secret-store project ID is metadata; a credential that authenticates
to that project is not.

## Start with the documentation contract

Every adopting repository needs three links in its README:

1. Its **variable inventory** in the project's documentation site: every consumed
   variable, required/default behavior, folder/tag, source, and read stage.
2. Its **runtime/command guide**: how development, operators, builds, and releases
   receive values, including override semantics and known integration gaps.
3. **This playbook** for reusable conventions.

Use the [environment inventory template](./environment-inventory.md). Keep actual
project key names in that project; do not copy its full table into this hub or
maintain a second copy in the README. The inventory should include optional,
test, signing, and deployment values, not just the minimum that boots an app.

## Mental model

| Piece                    | Responsibility                                                  | Where it lives                                                                                |
| ------------------------ | --------------------------------------------------------------- | --------------------------------------------------------------------------------------------- |
| Infisical                | Versioned values and access policies by project/environment     | Chosen cloud region or self-hosted instance                                                   |
| `.infisical.json`        | Project link and optional instance metadata, no credentials     | Committed                                                                                     |
| CLI                      | Fetches the selected values for one operation                   | One supported installation contract; see [tool doctor](../conventions/tool-doctor-pattern.md) |
| Login / machine identity | Authenticates the caller                                        | Local session or CI identity, outside app configuration                                       |
| Runtime schema           | Types and validates required values once delivered              | Existing code, such as a Zod env module                                                       |
| Inventory                | Explains each key's owner, consumers, defaults, and propagation | Project docs, linked from README                                                              |
| Adapter/wrapper          | Selects a source and consumer, bridges runtime boundaries       | Project tooling, covered by contract tests                                                    |

Keep the existing schema if it already works. An inventory documents that schema
and its other consumers; neither prose nor a tag filter replaces validation.

## Project and developer setup

1. Create one project per product and explicit environment slugs such as `dev`,
   `staging`, and `prod`. A UI display name such as "Development" is not the slug.
2. Inventory consumers and classify keys before importing values. Record which
   CI/release paths still use another store.
3. Set up folders, consumer tags, and identity permissions. Load values through an
   approved UI/CLI path that does not leave credentials in shell history or logs.
4. Commit `.infisical.json`. Select the correct instance/region; do not assume
   every organization uses the default US cloud endpoint.
5. Declare the CLI in the tool doctor. With a global-tool policy, do not also
   retain a competing workspace CLI that can shadow it during package scripts.
6. Developer flow: install dependencies, run the doctor for missing-tool guidance,
   `infisical login`, then rerun the doctor to validate access. An initial access
   failure before login is expected, not a dependency-install failure.

The doctor checks rather than installs. Verify its probe from the actual project
directory with a permitted narrow consumer; a readable project does not prove all
required keys exist or that the application receives them.

## Two axes: folders for risk, tags for consumers

Folders answer **"what authority or exposure does this value represent?"** Tags
answer **"which app or operation needs it?"** Each value has one home and can
have multiple consumers. Do not duplicate a credential merely because two apps
need it, and do not infer that two distinct authorities should share a credential
just because their environment-variable names match.

| Example folder | Intended contents / concern                                                     |
| -------------- | ------------------------------------------------------------------------------- |
| `/public`      | Non-credential configuration; not necessarily suitable for public publication   |
| `/database`    | Database credentials; scope actual roles to required access                     |
| `/cloudflare`  | Cloud/provider credentials; distinguish deploy authority from storage authority |
| `/auth`        | Session-signing or authentication secrets                                       |
| `/email`       | Mail-provider credentials                                                       |
| `/signing`     | Release signing/notarization material                                           |

These are examples, not a mandatory six-folder taxonomy. Split further when
permissions, owners, or rotation consequences differ. A one-key folder is useful
when it represents a genuinely different authority. Record a short description
of what a leak enables and where to rotate the credential.

Consumer tags should distinguish runtime and release stages, for example
`api-runtime`, `web-runtime`, `desktop-dev`, `database-tools`, `desktop-signing`,
and `deploy-api`. **Do not tag signing keys for ordinary desktop development.**
Libraries are generally not fetch consumers; the executable using them is.

### Selection is not authorization

`--tags api-runtime` limits a particular fetch. The logged-in identity may still
have authority to request other tags or paths. Configure environment/path/tag
restrictions in Infisical's permission policy and test denied reads as well as
allowed ones. Restrict who can change tags and values, too.

Development-scoped identities should not carry production authority. Credential
permissions at the actual database, storage, deploy, and signing providers matter
as much as Infisical read permissions. Renaming a folder does not narrow a token.

### Configuration is not automatically a secret

URLs, feature labels, bucket names, account IDs, and client ingestion keys can be
non-secret configuration. Keep constants in code when changing them should be
reviewed as code; use `/public` for centrally managed environment-specific values
when that operational choice is useful. Record which source was chosen.

Reading a config value may grant no authority while **writing** it changes access
or routing. An admin user-ID allowlist is the clearest example. Do not treat a
`/public` label as permission to publish user identifiers or grant everyone write
access. Never use a personal/admin analytics API key where an embedded client
ingestion key is expected.

Keeping root empty can make accidental non-recursive root exports return nothing.
It is not an access-control boundary: recursive unfiltered reads still traverse
the project. Require consumer selection and validate expected keys.

## Delivery depends on the consumer runtime

### Process environment

Use an explicit consumer filter:

```bash
infisical run --env=dev --recursive --tags api-runtime -- <command>
```

An explicit `--path` is also appropriate for an operation confined to one folder.
Recursive reads are acceptable **with a deliberate filter**, not as the default
way to hand an entire project to every process. Pin/test CLI behavior, including
imports and personal overrides, before relying on exact results.

Validate required keys after selection. A filter that matches nothing, or misses
one untagged required variable, need not cause a fetch error. A nonempty result
is not proof of a complete consumer contract. Reject unexpected duplicate names
when flattening multiple folders into one process namespace.

### Cloudflare Worker bindings

A Worker reads bindings, not arbitrary parent-process environment variables.
This applies both to direct `wrangler dev` and SSR with `@cloudflare/vite-plugin`.
Default Wrangler dev loading uses `.env`/`.dev.vars`; wrapping the Vite host
alone does not establish that server-side Worker values are present.

For a generated-file adapter, use this contract:

1. Fetch only the selected consumer's values with an explicit format.
2. On CLI versions such as `0.43.132`, `export` has no `--recursive`; enumerate
   folders and apply the same tag filter to each. Keep that list in one place.
3. Stage output in a unique owner-only temporary file in the destination
   directory. Do not truncate the active file before all fetches succeed.
4. Validate required keys, collisions, and expected scope without logging values.
5. Replace the destination atomically after success, clean up on failure, and
   define concurrent-writer behavior. `install -m 600` restricts permissions but
   is not by itself an atomic-rename or concurrency guarantee.
6. Detect competing legacy files. Wrangler can prefer `.dev.vars` over `.env`;
   Vite/Bun/mode-specific files can add their own precedence.
7. Verify a synthetic binding **inside the Worker**, including its Node
   compatibility surface if code reads `process.env` there.

A representative single-folder export is:

```bash
infisical export --env=dev --path=/database --tags api-runtime --format=dotenv --silent
```

It writes values to stdout: use only with synthetic fixtures or redirect through
the protected adapter. It is not a diagnostic command to paste into an issue.
The project must provide and test the adapter; this playbook does not imply any
named shell helper already implements all guarantees above.

### Build-time configuration and signing

Vite/electron-vite inline supported public prefixes at build time. Existing
`process.env` values normally take precedence over dotenv file values in Vite,
but prefix sets depend on the bundler/process. Verify the exact configured
prefixes and inspect a public sentinel in the output, not just the build exit.

- Builds consume an explicitly selected environment; do not hide a forced `dev`
  fetch inside a generic production-capable `build` command.
- Pass backend credentials to neither desktop/browser builds nor their dev
  processes. Never put a sensitive credential under an embedded public prefix.
- Supply signing secrets to the actual packaging/signing process. A completed
  child's injected environment does not propagate back to its parent or sibling.
- Document file transformations: a base64 Apple key in CI becomes a temporary
  `.p8` path, not a literal key string passed as a filename.
- Resolve and hash approved public config before a cacheable build. Fetching it
  inside a Turbo task is too late to affect that task's cache key. Track shared
  wrapper inputs and actual outputs, or disable the affected cache temporarily.
- Restarting an installed app cannot replace its compiled endpoint. Rebuild,
  publish, and update; runtime Worker values have a different propagation path.

## Precedence and explicit source selection

Infisical CLI `0.43.132` copies inherited values and then overwrites same-name
keys with fetched values. This differs from dotenvx's usual non-overriding mode.
`--secret-overriding` concerns personal versus shared Infisical secrets, **not**
preserving a shell-supplied `DATABASE_URL`.

Define an explicit supplied-environment mode for operator and CI commands, and
test both paths. A project's `SKIP_INFISICAL=1` convention is one possible adapter
interface, not an Infisical CLI feature. Document the exact syntax that exists
in that project and validate the required consumer keys in either mode.

`CI` should describe execution context, not ambiguously select a database target.
If a wrapper treats every nonempty value as true, `CI=false` and `CI=0` also
bypass fetching; document that actual behavior until corrected. Never suggest a
production override that silently changes target. Verify the intended database
identity with a non-mutating operation before any operator write.

## CI and migration boundaries

PR checks should remain credential-free when their purpose only needs placeholders
or synthetic fixtures. Their success does not validate a live secret integration.
Production releases can use OIDC with scoped machine identities instead of a
long-lived bootstrap secret. Follow the current
[Infisical GitHub Actions documentation](https://infisical.com/docs/integrations/cicd/githubactions),
pin the reviewed action revision, and verify its path/filter behavior rather
than assuming CLI tags map identically to action inputs.

OIDC trust must bind the intended repository and deployment context. A job using
a GitHub Environment normally has subject
`repo:<owner>/<repo>:environment:<name>`, replacing the ordinary branch subject.
Workflows sharing that Environment share that subject; use separate environments
or supported additional claims/policies if workflow-level isolation is required.
Configure audience/issuer against the provider's actual expected claims and test
rejected contexts, not just the successful token exchange.

**Do not remove existing GitHub Secrets until each replacement release path has
been exercised successfully.** Document GitHub as the current production source
until then. A GitHub-to-GitHub token such as release-please can stay in GitHub;
routing it through another service adds a bootstrap dependency without a useful
ownership improvement.

## Migration and maintenance order

1. Inventory keys and read sites: schemas, inline reads, optional defaults,
   examples, scripts, workflows, and signing tools. Never import actual values
   into documentation or generated reports.
2. Choose folders/tags and policies, link the project, declare tooling, and
   publish the README links and inventory before claiming migration completion.
3. Preserve legacy local configuration deliberately in protected, ignored storage
   while validating. Detect conflicts; do not silently delete developer files.
4. Migrate one entry point at a time. Validate a meaningful app operation, not
   merely the fetch or boot. Include clean-checkout and stale-file cases.
5. Test empty tags, missing required keys, wrong target, interrupted export,
   duplicates, concurrent writes, secret scope, and cache invalidation with fake
   values. Include shared scripts in relevant CI path filters.
6. Migrate releases as a separate milestone, documenting representations and
   runtime/build/signing lifetimes. Verify before removing the previous source.
7. Remove obsolete examples/backups only after validation, update the inventory,
   and mark historical plans/audits with their reviewed revision.

Whenever a variable changes, update its one inventory row, schema/read site,
tag assignment, any exporter folder list, and applicable release mapping. Move
then verify tags: a move operation may change assignments. Untagged keys should
be found by inventory validation rather than silently disappearing from consumers.

## Rotation and incident handling

- Record the authoritative store, provider owner, consumers, and required
  restart/rebuild/redeploy action. A store edit alone is not a completed rotation.
- Avoid combining transport migration with credential rotation unless an incident
  requires it; otherwise failures are difficult to attribute.
- Revoking Infisical access does not invalidate exported provider credentials or
  running processes. A lost laptop can require rotating multiple credentials,
  not just revoking its Infisical session.
- Review audit/version-history availability for the chosen plan and retention
  requirements. Do not assume a pricing-tier promise is an enduring guarantee.
- `--silent` suppresses routine messages; it is not a redaction guarantee. Do not
  record verbose real-secret CLI sessions, full environments, or exported files
  in diagnostic artifacts.

## References and adoption example

- [Environment inventory and README contract](./environment-inventory.md)
- [Tool doctor](../conventions/tool-doctor-pattern.md)
- [CLI overview](https://infisical.com/docs/cli/overview)
- [CLI 0.43.132 source — precedence and flags](https://github.com/Infisical/cli/blob/v0.43.132/packages/cmd/run.go)
- [Machine identities](https://infisical.com/docs/documentation/platform/identities/machine-identities)
- [Varlock evaluation — optional, separate](./varlock-evaluation.md)
- [Kaipu project inventory](https://github.com/csdev19/kaipu-record-monorepo/blob/main/apps/documentation/src/content/docs/deployment/secrets-layout.md)
  applies this pattern. On the reviewed local branch, tag-scoped development is
  implemented; its documented gaps and pending production migration remain
  project-specific. Check the target branch before assuming the linked main
  reference contains the latest inventory update.
