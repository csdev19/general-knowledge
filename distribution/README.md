# Distribution — shipping a desktop app outside the store

_How a desktop app becomes a signed, notarized download that updates itself — the
stages that are the same in every toolchain, and the traps that are not._

A monorepo playbook already covers **when** a release runs and **what** triggers it:
[monorepos/ci-cd-pipelines.md](../monorepos/ci-cd-pipelines.md) for tag-driven pipelines and
[monorepos/release-please-playbook.md](../monorepos/release-please-playbook.md) for cutting the
tag itself. Those docs are toolchain-agnostic on purpose. This folder is the other half: the
part that is **specific to the thing being built**, and the part that decides whether the result
looks like a product or like a build artifact.

---

## The spine — identical in every toolchain

macOS gives you no choice about the order. Each step consumes the output of the previous one, and
doing two of them out of order silently produces an artifact that fails on a user's machine and
passes on yours.

| # | Stage | What it produces | Skipping it costs |
| --- | --- | --- | --- |
| 1 | Build + **Developer ID** sign, Hardened Runtime on | `.app` | Gatekeeper refuses to open it |
| 2 | **Notarize** the app, then **staple** the ticket | `.app` with a stapled ticket | First launch needs network; offline users see a scary dialog |
| 3 | Package the **DMG** (app + `/Applications` symlink + window design) | `.dmg` | A bare `.app` download that users run from `~/Downloads` forever |
| 4 | Sign, notarize and **staple the DMG too** | distributable `.dmg` | The disk image itself is quarantined even though the app inside is clean |
| 5 | Build the **update archive** and sign the feed | `.zip` + `appcast.xml` (or equivalent) | No update path; every fix is a manual reinstall |
| 6 | **Publish**: immutable artifacts → stable alias → live feed | public URLs | A feed that points at objects that do not exist yet |

Two rules that come out of stage 6 and are worth stating on their own:

- **The live feed is the release boundary.** Upload immutable versioned objects first, the stable
  `latest` alias second, the feed last. A failure before the feed leaves every installed user on
  the previous version, which is a safe state. A failure after it does not.
- **Versioned objects are immutable.** A rerun that produces different bytes for the same version
  must fail loudly, not overwrite. The fix for a bad build is a higher version, never a new upload
  under the old one.

---

## The part the toolchain does not help with

Signing has a manual. The icon and the install window do not, and they are the entire first
impression: they are what the user sees before a single line of your code runs.

- [macos-app-icon-and-dmg.md](./macos-app-icon-and-dmg.md) — the app icon's real geometry, and how
  to make Finder open the DMG with your layout, headlessly, in CI. Toolchain-agnostic: the icon is
  PNG slots and the DMG is `hdiutil` + a `.DS_Store` whether the app is Swift, Rust or JavaScript.

## Per-toolchain challenges

The spine above is fixed. What changes is where the version number comes from, what the build
system will and will not generate for you, and which tool holds the signing switches.

| Doc | Toolchain | Status |
| --- | --- | --- |
| [swift-xcode-macos.md](./swift-xcode-macos.md) | Swift / Xcode / Sparkle | Written — drawn from a shipped app, five releases (`v0.1.1` → `v0.4.0`) |
| `rust-tauri-macos.md` | Rust / Tauri 2 | Not written yet |
| `electron-macos.md` | Electron / electron-builder | Not written yet |

The two unwritten docs are slots, not promises: write each one from a repository that has actually
released, the way the Swift doc was, rather than from the tool's documentation.
