# Stack: Desktop Tauri

_Local-first Tauri 2 desktop app: Rust core that owns storage and native chrome, a React
webview that runs the shared TypeScript use cases, and typed commands between them._

## When to use it

When the app must be small and light at idle (system webview, no bundled Chromium) and its heavy
work is not a Chromium-specific media pipeline. Read the framework ADR first.

## Reading list (in order)

**1 · Framework decision**
- [Electron vs Tauri (ADR)](../desktop/electron-vs-tauri.md) — when each fits, where the real gap is

**2 · Tauri architecture**
- [Where business logic and data live](../desktop/tauri-architecture.md) — webview vs Rust
  placements, the hybrid default, typed commands, testing at the wire
- [Client state persistence](../architecture/client-state-persistence.md) — which side owns a
  persisted setting (here: the Rust core; the webview mirrors it)
- [Distribution](../distribution/README.md) — signing, notarization, DMG, update feed

**3 · Shared foundations**
- [DDD + hexagonal architecture](../architecture/README.md) — the Tauri adapter is one more
  implementation of a repository port
- [`infra-*` packages](../packages/infrastructure-naming.md), [monorepo](../monorepos/monorepo-structure.md)

## Assembly notes (specific to this stack)

- **Two languages, one contract.** Generate TypeScript bindings from the Rust commands, commit
  them, and fail CI when they drift.
- **One React.** Workspace packages can resolve their own React; set
  `resolve.dedupe: ["react", "react-dom"]` in Vite or the window renders blank.
- **Fixed dev port.** Vite must serve on the port in `build.devUrl` (`strictPort`).
- **Renderer build first.** The Rust build embeds `frontendDist`; build the renderer before
  compiling release binaries.
- **Updater keys are not Apple keys.** `tauri-plugin-updater` verifies bundles with its own
  minisign key (`tauri signer generate`). Keep `createUpdaterArtifacts` off locally and turn it on
  in the release job, so a local bundle does not need the private key.
- **Linux CI** needs the WebKitGTK dev packages even to compile tests.
