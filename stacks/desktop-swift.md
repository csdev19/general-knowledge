# Stack: Desktop Swift (native macOS)

_Native SwiftUI macOS app: feature modules as SwiftPM targets with compiler-enforced DDD
layering, a menu bar overlay shell, and direct distribution as a signed, notarized DMG with
Sparkle updates._

**Template:** [niway-dev/swift-desktop-template](https://github.com/niway-dev/swift-desktop-template)
— "Use this template", then `scripts/customize.sh --name <App> --bundle-id <id>`.

## When to use it

macOS only, and you want the smallest, most native app: menu bar utilities, overlays, anything
that lives next to the system rather than in a browser engine. Pick
[desktop-tauri](./desktop-tauri.md) or [desktop-electron](./desktop-electron.md) when the app must
also run on Windows/Linux or share a TypeScript domain with web and mobile.

## Reading list (in order)

**1 · Architecture**
- [DDD layering in Swift with SwiftPM modules](../desktop/swift-modular-architecture.md) —
  per-feature targets, which parts of hexagonal to keep, the Swift details that make it hold
- [The AppKit lifecycle under SwiftUI](../desktop/swift-macos-app-lifecycle.md) — the
  composition root builds, the app delegate acts; why an AppKit side effect taken in the `App`
  initializer silently stops mouse delivery, and the probe-build diagnosis for that class of failure
- [Domain-layer contracts](../architecture/domain-layer-contracts.md) and
  [repository pattern](../architecture/repository-pattern.md) — the ports, language-agnostic
- [Bounded contexts](../architecture/bounded-contexts-complete-guide.md) — when a second feature
  needs data from the first

**2 · Distribution**
- [Distribution spine](../distribution/README.md) — sign → notarize → staple → DMG → feed → publish
- [Swift / Xcode + Sparkle](../distribution/swift-xcode-macos.md) — the traps, as symptoms
- [App icon and DMG window](../distribution/macos-app-icon-and-dmg.md)
- [release-please playbook](../monorepos/release-please-playbook.md) and
  [Infisical secrets](../infra/infisical-secrets.md)

## Assembly notes (specific to this stack)

- **No monorepo tooling.** One repo, one app; `Modules/` is a local SwiftPM package, not a
  workspace. The shared base's Turborepo/Bun items do not apply.
- **XcodeGen is required locally and in CI** (`brew install xcodegen`); the `.xcodeproj` is
  generated and ignored.
- **Tests run without Xcode:** `swift test --package-path Modules`. The release workflow runs
  them first; there is no per-PR macOS job by default (macOS minutes cost ×10).
- **Updates are off until configured.** A fresh app has no Sparkle key or feed URL and runs
  normally without them; the first-release checklist is in the template's
  `docs/release-runbook.md`.
- **The DMG window design is per app.** The template ships a plain DMG; the committed
  `.DS_Store` stores the volume name and icon positions, so capture it after the app has its
  final name.
- **One instance per bundle id.** `LSMultipleInstancesProhibited` makes a second copy started
  from Xcode exit at once with *Finished running*. Build probes under a separate bundle id
  (`PRODUCT_BUNDLE_IDENTIFIER=<id>.probe`; never override `PRODUCT_NAME` on the command line),
  and never launch the app from an agent session while the developer runs it from Xcode.
- **Click smoke test.** Keep a `scripts/click-smoke.sh` (build under a smoke id, launch by pid,
  one synthetic click on a window, grep the `input` log, exit 1 on silence) and run it before
  merging changes to windows, events or launch. It is the only automated check that catches
  silent AppKit input failures — see the lifecycle page.
- **Sandbox and permissions.** Mouse-down global monitors work sandboxed without a prompt; key
  monitors need accessibility, and the supported sandboxed path for keys is `CGEventTap` with
  Input Monitoring. Verify permission behaviour on a Mac that has never seen the app.
