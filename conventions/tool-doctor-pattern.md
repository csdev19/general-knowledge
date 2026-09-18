# Tool doctor: declare global dependencies, never install them

> Status: **accepted.** First applied in `kaipu-record-monorepo` (2026-09-18)
> as `bun run setup`.

`bun install` handles everything in `package.json`. It does nothing about the
tools that must already exist on the machine — a secrets CLI, a cloud CLI, a
language runtime, a database client. Those are invisible until something fails
with a message that has nothing to do with the real problem.

The pattern: **one script that checks and reports. It never installs.**

## The rule

> A setup script tells you what to install and how. It does not install it.

Installing looks helpful and costs more than it returns:

- **It is a supply-chain surface.** A script that downloads a binary runs before
  you have read it. A fallback that curls a release tarball has **no checksum to
  verify against** — precisely the protection the package manager it replaces
  was providing. That was a real flaw in this pattern's first draft.
- **It cannot keep up.** Install commands differ per OS, per architecture, per
  package manager, and change over time. A checker only needs to answer "is it
  here, is it new enough" — which is stable.
- **It hides the decision.** You should know what landed on your machine and
  from where.

The exception worth naming: an ephemeral environment you rebuild constantly — a
devcontainer, a CI image — where the install is itself reviewed configuration.
There, install. On a developer's machine, report.

## What the script checks

Three things, in order, because each makes the next meaningful:

1. **Presence** — is the binary on `PATH`?
2. **Version** — is it new enough? A too-old tool fails in a way that looks like
   a bug in your code.
3. **Access** — can it actually do the job?

The third is the one people skip and the one that pays. A secrets CLI can be
installed and up to date and still be logged out, or pointed at a project you
lack access to. Prove it with one real operation:

```bash
if infisical secrets --env=dev --recursive --silent >/dev/null 2>&1; then
  ok "can read the dev environment"
else
  fail "cannot read the dev environment — run: infisical login"
fi
```

**One check that proves the chain beats two that can disagree.** The first draft
here probed for a login session *and* read a secret, and cheerfully printed
"not logged in" directly above "can read the dev environment" — the session
probe used the wrong command. A check that can be wrong in a way the real
operation contradicts is worse than no check.

## Shape

Declare the tools as data so the list reads as documentation:

```bash
# name | min version | why it is needed | install command
REQUIRED=(
  "bun|1.3.4|runs every script and the workspace install|https://bun.sh"
  "infisical|0.40.0|every dev and db script fetches secrets|brew install infisical/get-cli/infisical"
)
OPTIONAL=(
  "gh||opening and reviewing pull requests|brew install gh"
)
```

Output when something is missing:

```
Required tools
  ✓ bun          1.4.2
  ✗ infisical    missing
       needed for: every dev and db script fetches secrets
       install:    brew install infisical/get-cli/infisical

1 tool(s) missing. Install them with the commands above, then re-run.
```

Non-zero exit, so CI and pre-flight hooks can depend on it.

Details that matter:

- **Say why each tool is needed.** "Install infisical" invites skipping; "every
  db script fetches secrets from it" does not.
- **Distinguish required from optional.** Optional tools are reported and do not
  fail the run.
- **Parse versions leniently.** Tools disagree about where the number sits —
  take the first version-shaped token on the first line and compare with
  `sort -V`.
- **Make it idempotent and fast**, so re-running it is free and people do.
- **Test the failure path**, not just the happy one. Run it with a stripped
  `PATH` and read the output as a newcomer would.

## Wire it in

```json
"setup": "bash scripts/setup-dev.sh"
```

Point the README's first-run section at `bun run setup` rather than a list of
tools in prose — prose goes stale silently, the script fails loudly.

Resist calling it from `postinstall`: `bun install` runs constantly, and a
check that interrupts it gets muted rather than fixed.

## Why not just a README section

A README lists tools and drifts. The script is executable documentation: when a
tool is added or a minimum version rises, the check fails on the next run and
the fix is one line in the array. Nobody has to notice a paragraph went stale.

## Related

- [Infisical secrets playbook](../infra/infisical-secrets.md) — the first tool
  this pattern was built to cover.
