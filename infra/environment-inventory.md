# Environment inventory and README contract

> Status: **reusable documentation convention.** Last reviewed: 2026-09-18.
> Companion to the [Infisical playbook](./infisical-secrets.md); no Varlock dependency.

The README gets a developer started. The project's documentation explains every
variable. The shared playbook explains the reusable delivery and security rules.
Keep those responsibilities linked instead of copying the same tables everywhere.

## Required README section

Include the supported runtime/CLI, how project access is granted, installation,
login, and an access check. Link to one canonical project inventory and the
runtime/command guide. Name any remaining migration boundary explicitly.

Adapt this example to **commands and paths that actually exist**:

```markdown
## Environment setup

Install the supported runtime and Infisical CLI, then obtain access to this
project's development environment. The setup checker reports missing tools;
it does not install them.

1. Install workspace dependencies.
2. Run the project's setup checker for tool guidance.
3. Run `infisical login`, then rerun the checker to verify project access.
4. Start the documented development command.

Do not create or edit `.env` files by hand. Migrated commands fetch their values;
Worker files are generated. Plain builds consume their supplied environment.

- [Project variable inventory](<project-docs-inventory-path>) — keys, folders/tags,
  required/default behavior, consumers, delivery stages, and integration status.
- [Runtime and operator guide](<project-docs-runtime-path>) — invocation and overrides.
- [Reusable Infisical playbook](https://github.com/csdev19/general-knowledge/blob/main/infra/infisical-secrets.md).

Production currently uses <actual source>. Changing a build-time value requires
a rebuild; changing a deployed binding requires the documented deployment step.
```

The links are placeholders for the adopting project, not proposed file paths for
this hub. Do not say "all values come from Infisical" when release workflows or
legacy entry points still use GitHub, ambient values, or another source.

## Inventory content

Use the project's established documentation site, not a second app-local note.
Record a review date and source revision. Separate **implemented source wiring**,
**intended mappings**, and **live-store verification**; code inspection cannot
prove a dashboard key exists or has the right tags.

### Canonical key table

| Field                 | What to record                                                                                                   |
| --------------------- | ---------------------------------------------------------------------------------------------------------------- |
| Exact key             | Consumer spelling, including any bundler prefix                                                                  |
| Classification        | Credential, non-secret config, security-sensitive config, invocation control, platform binding, or derived alias |
| Owner / folder        | System where it is issued/rotated and its Infisical location, if applicable                                      |
| Consumer / tag        | App or operation; distinguish runtime, tests, build, signing, and deploy                                         |
| Requirement           | Always required, feature-required, optional, or obsolete                                                         |
| Format                | URL, comma-separated IDs/origins, project ingestion key, file path, base64 material, etc.                        |
| Missing behavior      | Actual default, skipped test, disabled feature, initialization failure, or later runtime failure                 |
| Source by environment | Infisical dev, GitHub production, committed literal, runner-derived value, etc.                                  |
| Read stage            | Process startup, Worker runtime, compiled bundle, packager/signing, or deploy step                               |
| Propagation           | Next command, restart, regenerate file, rebuild, redeploy, or publish/update                                     |
| Evidence              | Schema/read site/script/workflow, plus actual validation status                                                  |

Split large tables by consumer or stage for readability. A shared credential
should still have one canonical definition; reference it from other consumers.
Describe value formats with placeholders or non-sensitive examples, never real
credentials or copied secret exports.

### Cover more than boot-time required values

- Optional mail, analytics, feature-gate, public URL, and sender configuration.
- Test database URLs and destructive test behavior; distinguish skipped tests
  from successful verification.
- Operator target selection and precedence controls.
- Signing/deployment keys, file conversions, and the process that consumes them.
- CI aliases: e.g. an R2 secret mapped to `AWS_SECRET_ACCESS_KEY` does not need a
  second stored secret just because the tool expects another name.
- Platform resources such as service/static-asset bindings, which are not dotenv
  strings fetched from Infisical.
- Legacy tooling whose scripts still exist, with its status rather than an
  implicit instruction to provision every old variable.

Do not attempt to inventory every environment variable a dependency or OS can
read. Include project-configured controls and actual app/script/workflow inputs;
identify standard runner/bundler facilities separately.

## How to collect the inventory

1. Read runtime schemas, then direct/dynamic `process.env`, `import.meta.env`,
   and Worker binding accesses. Schemas alone miss inline optional config.
2. Read package scripts, shared wrappers, bundler configs, CI jobs, and packaging
   scripts. Follow aliasing, file decoding, and subprocess boundaries.
3. Compare against example files and the previous inventory; examples are clues,
   not authority. Never read real local env files just to list known key names.
4. If administrative access is needed, request key names, tags, policy scopes,
   and metadata without values. Do not equate intended tags with actual RBAC.
5. Record mismatches explicitly: missing fetch wiring, old paths, optional keys
   not provisioned, or a schema that still requires a retired credential.
6. Validate doc links, names, and a relevant docs build. Runtime claims need
   separate synthetic or authenticated smoke-test evidence.

## Maintenance trigger: a variable changes

The same change should update the key's inventory row, declaration/read site,
consumer tag mapping, required-key validation, exporter folder list if needed,
and release mapping if applicable. Verify that a new credential does not enter
a public build or ordinary desktop development scope.

Moving a value is not complete until tags and consumers still resolve correctly.
Rotating it is not complete until the documented propagation action is done.
Removing it is not complete until callers and deployment aliases stop requiring it.

Leave the README as a stable entry point; do not make reviewers synchronize a
second variable table there. Historical audit evidence stays tied to its original
revision; current reference docs say what works now and what remains pending.

## Completion criteria

- A new developer can reach the canonical inventory from the README in one click.
- Every project-consumed key is described, including defaults and read stage.
- No real credentials, exported files, or environment-specific user IDs appear.
- Production migration status and known gaps match the reviewed branch.
- Shared guidance links here; project keys are not copied into general-knowledge.
- Varlock evaluation is linked as an optional separate topic, not required setup.
