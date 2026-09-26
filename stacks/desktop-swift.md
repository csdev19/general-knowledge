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
