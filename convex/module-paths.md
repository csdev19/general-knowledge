# Convex module paths: no hyphens under `convex/`

_Every file under `convex/` becomes a deployed module, and module paths only accept
alphanumerics, underscores and periods. Nothing local checks that; the first push to a
real deployment does._

## The failure

```
InvalidConfig: Invalid module path '_lib/employee-events.js': Path component
employee-events.js can only contain alphanumeric characters, underscores, or periods.
```

`convex dev` / `convex deploy` bundles **every** `.ts` / `.js` file under `convex/`
except `_generated/`. That includes the places that look private:

- underscore-prefixed helper directories (`_lib/`, `_utils/`);
- test directories (`__tests__/`) and `*.test.ts` files living next to functions.

So one hyphen in any file or directory name under `convex/` rejects the whole push.
Observed with Convex CLI 1.32.

## Why nothing catches it first

`tsc`, lint, and Vitest with `convex-test` all run against the source tree and are happy
with `org-access.ts`. The module-path check exists only on the deployment side, at push
time. A repository can therefore be fully green — type-check, tests, the whole `verify`
pipeline — and still be undeployable, and it finds out the first time someone connects a
real deployment.

That first connection is often deferred on purpose. An acceptance criterion phrased as
"would start against a dev deployment when one exists (not required now)" is exactly the
deferred check that lets this class of bug ship: it is honest about what was not run, and
it still leaves the failure for later.

## The rule

- Name every file and directory under `convex/` in **camelCase** (or with underscores):
  `employeeEvents.ts`, `orgAccess.test.ts`, `_lib/orgFunctions.ts`.
- Kebab-case stays fine everywhere else in the repository (`domain`, `application`, the
  web app). The constraint is Convex's, so it applies only inside the folder Convex bundles.
- The module path is also the public function path (`api.employeeEvents.list`), which is
  one more reason camelCase reads naturally there.

## Enforce it before a deployment exists

Give the repository one guard that runs in the normal test or `verify` pipeline, so the
rule holds without credentials. A small Vitest test is enough:

```ts
// convex/__tests__/modulePaths.test.ts
import { readdirSync } from "node:fs";
import { join, relative } from "node:path";
import { expect, test } from "vitest";

const root = join(import.meta.dirname, "..");
const invalid = /[^A-Za-z0-9_.]/;

function walk(dir: string): string[] {
  return readdirSync(dir, { withFileTypes: true }).flatMap((entry) => {
    if (entry.name === "_generated") return [];
    const path = join(dir, entry.name);
    return entry.isDirectory() ? [path, ...walk(path)] : [path];
  });
}

test("every path component under convex/ is a valid Convex module name", () => {
  const offenders = walk(root)
    .map((path) => relative(root, path))
    .filter((path) => path.split("/").some((part) => invalid.test(part)));
  expect(offenders).toEqual([]);
});
```

(Illustrative; adjust the root if `convex/` lives in a package such as `packages/<backend>`.)
A lint step in `verify` that runs the same walk works equally well. What matters is that
the check replaces "will be caught when a deployment exists" with a check that runs on
every push, credential-free.

## Measured example

From the owner's HR operations platform: a six-block phase shipped in nine stacked PRs with
293 passing tests. The first push to a real deployment failed on three hyphenated files —
`_lib/employee-events.ts`, `_lib/org-functions.ts` and `__tests__/org-access.test.ts`. The
fix was a rename plus the matching imports; the guard test above is what keeps it fixed.

## Related

- [Convex client connection](./client-connection.md) — first-time deployment, where this
  failure surfaces.
- [Better Auth in Convex](./better-auth.md) — the deployment env a first push also needs.
- [Infisical secrets playbook — Convex deployment env](../infra/infisical-secrets.md#convex-deployment-env)
  — how secrets reach a Convex deployment.
