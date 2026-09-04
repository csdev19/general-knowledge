# Desktop (Electron) Knowledge Hub

Reusable architecture patterns for building a local-first desktop app with Electron: main/renderer/preload split, a type-safe IPC contract, native OS permissions, a filesystem-first storage vault, a heavy media/compute pipeline, distribution, and feature flags. Generalized from a real screen-capture app — product names stripped, patterns kept.

## Architecture

- [electron-vs-tauri.md](./electron-vs-tauri.md) — ADR: why a media-heavy app can stay on Electron; the real perf gap is the capture/encode pipeline, not the framework, plus the triggers for reconsidering Tauri.
- [main-process-architecture.md](./main-process-architecture.md) — How the Electron main process boots, the `register*` subsystem pattern, window/tray lifecycle, and the one-renderer-two-windows trick.
- [renderer-architecture.md](./renderer-architecture.md) — Renderer folder layout (pages/features/ui/shell), the three-step legacy-screen migration workflow, and the CSS-tokens + `cx()`/`data-*` house style.
- [product-philosophy.md](./product-philosophy.md) — Local-first product principles: the artifact is the product, cloud is a capability, hide successful infrastructure, and the three-axis state model.

## IPC & process model

- [ipc-contract.md](./ipc-contract.md) — The main↔renderer bridge: `contextBridge` preload, channels grouped by subsystem, `invoke` vs `send`, and the TypeScript-verified shared types as the source of truth.
- [media-pipeline.md](./media-pipeline.md) — A heavy media/compute pipeline in Electron: engine as a module-singleton outside React, positional disk writes, cross-window state via a main-process hub, hardware encode, and robust failure recovery.

## Native integration

- [permissions-and-onboarding.md](./permissions-and-onboarding.md) — The OS gates behind capture (mic, screen, **system audio**, camera) per platform and which ones Electron can actually check; the macOS backend choice that decides which permission system audio needs; dev-mode TCC attribution to the terminal; re-read-on-focus and "the attempt is the truth"; the first-run onboarding flow; and how to prove it all with a probe and a self-skipping E2E.
- [library-vault.md](./library-vault.md) — Local-first storage: files as the source of truth, JSON metadata sidecars, a configurable vault directory, and a privileged streamable media protocol for playback.

## Distribution & product

- [filesystem-first-monetization.md](./filesystem-first-monetization.md) — ADR: why filesystem-first (atomic sidecars, metadata inside the vault) and why local-first doesn't hurt monetization — sell the network/cloud layer, never local features.
- [analytics-and-flags.md](./analytics-and-flags.md) — Analytics architecture (renderer-primary + main-sink), offline-safe feature flags, per-device identity, the two-channel error rule, and a privacy posture for a screen-capable app.
