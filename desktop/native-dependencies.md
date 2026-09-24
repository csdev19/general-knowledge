# Native dependencies in an Electron app: rebuild explicitly, never in `postinstall`

A native (N-API) module ships prebuilt binaries for **Node's** ABI. Electron bundles a
different Node with a different ABI, so the prebuild that satisfies `node` does not
satisfy `electron`. The module must be rebuilt against the Electron you actually run.

The tempting fix is `"postinstall": "electron-builder install-app-deps"`. Do not. It is
the same rule as the [tool doctor](../conventions/tool-doctor-pattern.md), applied to
compilation instead of installation: **`bun install` is not allowed to compile code,
download binaries, or run anything nobody read.**

## Why `postinstall` is the wrong home

1. **It breaks CI everywhere the dep is irrelevant.** A monorepo's web and API jobs run
   `bun install` too. On a Linux runner with no X11 development headers the build dies:

   ```
   ../libuiohook/src/x11/input_helper.c:23:10:
     fatal error: X11/keysym.h: No such file or directory
   ```

   Workflows that never touch the desktop app now fail on `bun install --frozen-lockfile`.

2. **It is the shape of a supply-chain attack.** A lifecycle script that compiles and
   executes on every install is the entry point recent npm compromises use. Your own
   script is not that attack; holding the rule uniformly is what makes an exception
   visible when it appears.

3. **It runs constantly and silently.** Most installs do not need a rebuild.

## The pattern

**An explicit script**, called by every path that needs it:

```jsonc
// package.json
"rebuild:native": "./scripts/rebuild-native.sh"
```

```yaml
# electron-builder.yml — packaging must not try to rebuild on its own
npmRebuild: false
```

**A stamp so staleness is detectable**, written by the script after a successful rebuild:

```bash
# scripts/rebuild-native.sh (sketch)
electron_version=$(node -p "require('electron/package.json').version")
npx electron-rebuild            # or electron-builder install-app-deps
echo "$electron_version" > node_modules/.native-deps-electron-version
```

The stamp lives **inside `node_modules`** on purpose: a fresh install wipes it, and an
Electron upgrade makes it mismatch. Both are exactly the moments a rebuild is due.

**The doctor reports it, and does not do it.** The setup script compares the stamp with
the installed Electron and prints the command:

```
✗ native deps built for Electron 38.2.1, installed 39.8.10
  → bun run rebuild:native
```

**Every packaging path calls it explicitly** — `build:mac`, `build:win`, `build:linux`,
the unpacked build, and the release workflow. A release must not depend on someone
remembering.

## Degrade, do not crash

Design the consumer so a missing or ABI-mismatched binary is a *feature being off*, not a
failed launch:

```ts
try {
  hook = require("uiohook-napi");
} catch {
  hook = null;            // the feature is simply unavailable
}
```

That silence is precisely why the doctor check has to exist: without it, the only symptom
is a feature that quietly does nothing, and nobody connects it to an install.

## Checklist

- [ ] No `postinstall` anywhere in the repo.
- [ ] `rebuild:native` script + `npmRebuild: false`.
- [ ] A stamp under `node_modules`, compared by the setup doctor.
- [ ] Every packaging and release path calls the rebuild explicitly.
- [ ] The consumer degrades to "feature off" on a load failure.
- [ ] CI for unrelated apps does not compile the native dep.

## Related

- [[tool-doctor-pattern]] — check, never install; this is its compilation counterpart.
- [[ci-cd-pipeline-strategy]] — why an install-time compile is a cross-app CI cost.
