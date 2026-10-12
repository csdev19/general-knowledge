# Swift tooling: format, lint, dead code and the verify gate

**Use the toolchain's own `swift format` for both formatting and lint. Let the Swift 6
compiler be the main linter, and make its warnings errors in one gate script.** Run that gate
in a lefthook pre-push hook and, when a PR backstop is worth the macOS minutes, in CI. Keep
Periphery (dead code) and the sanitizers as manual checks. That covers a native macOS app with
no extra dependency in the build and one brew install per machine.

This page is the Swift application of
[ci-cd-pipeline-strategy.md](../conventions/ci-cd-pipeline-strategy.md): one `verify` gate, the
same script locally and in CI, the expensive work on tags.

## The choices, and the strongest case against each

| Concern | Choice | Rejected | The case against the choice |
| --- | --- | --- | --- |
| Format | `swift format` (swiftlang/swift-format), shipped with the toolchain since Swift 6 | nicklockwood/SwiftFormat | SwiftFormat has more rules and more control (`redundantSelf`, import sorting) and is faster. It costs a brew dependency and a second config dialect. |
| Lint | the compiler in Swift 6 language mode, plus `swift format lint --strict` | SwiftLint | SwiftLint's semantic rules (force unwraps, unused closure parameters, weak delegates) catch what swift-format does not, and it is the most adopted linter. Add it later via `SimplyDanny/SwiftLintPlugins` with a short opt-in list if those bugs appear. |
| Warnings | errors in the gate only (`SWIFT_TREAT_WARNINGS_AS_ERRORS=YES`, `-Xswiftc -warnings-as-errors`) | errors in the Xcode project | An Xcode upgrade that adds deprecations turns the gate red overnight. That is the point: fix them while they are few. Keeping them warnings in the IDE lets experiments build. |
| Dead code | Periphery, by hand, about monthly and before a release | Periphery in the gate | Without the gate, dead code lingers between scans. Its findings need judgement (SwiftUI previews, `public` API used by another scheme, `Hashable` keys), so a gate would grow an exclusion list instead. |
| Hooks | lefthook (`brew install lefthook`), the same tool as the web repos | `.githooks` + `core.hooksPath` | Hooks are skippable and per machine; that is what the CI backstop is for. |

## Setup

**1. Config.** Start from the defaults and change only what the code already does:

```bash
swift format dump-configuration > .swift-format
# then set "indentation": {"spaces": 4} and "lineLength": 140
```

The default is 2 spaces. Linting 4-space code against it buries every real finding under
indentation noise.

**2. One mechanical commit.** Reformat everything in a commit that contains nothing else,
and list it so blame skips it:

```bash
swift format --in-place --recursive <app> <app>Tests <package>/Sources <package>/Tests
echo "<sha>" >> .git-blame-ignore-revs
git config blame.ignoreRevsFile .git-blame-ignore-revs   # once per clone
```

Then fix what lint still reports in separate commits. Usually that is
`TypeNamesShouldBeCapitalized` on the Xcode template types (`<app>_appApp`), and renaming the
type is enough; the file can keep its name.

**3. The gate, `scripts/verify.sh`:**

```bash
#!/usr/bin/env bash
set -euo pipefail
cd "$(dirname "$0")/.."
sources=(<app> <app>Tests <app>UITests <package>/Sources <package>/Tests)

swift format lint --strict --recursive "${sources[@]}"
swift test --package-path <package> -Xswiftc -warnings-as-errors
xcodebuild -project <app>.xcodeproj -scheme <app> -configuration Debug \
    -destination 'platform=macOS' -derivedDataPath build/DerivedData \
    CODE_SIGNING_ALLOWED=NO SWIFT_TREAT_WARNINGS_AS_ERRORS=YES \
    -quiet build-for-testing
```

- `build-for-testing` compiles the app and its test targets without launching anything, so it
  works headless and unsigned. `xcodebuild test` on an unsigned app target can fail with
  "Could not launch" for reasons unrelated to the code.
- Keep logic in the SwiftPM package, so `swift test` covers it in seconds without an app host.
  See [swift-modular-architecture.md](./swift-modular-architecture.md).
- `-destination 'platform=macOS'` silences the "multiple matching destinations" warning.

**4. Hooks, `lefthook.yml`:**

```yaml
pre-commit:
  commands:
    format:
      glob: "*.swift"
      run: swift format --in-place {staged_files}
      stage_fixed: true
pre-push:
  commands:
    verify:
      run: scripts/verify.sh
```

**5. Periphery, `.periphery.yml`.** Run it with `periphery scan`:

```yaml
project: <app>.xcodeproj
schemes:
  - <app>
retain_public: true                 # the package's public API is used by other schemes
retain_assign_only_properties: true # Hashable cache keys store fields only to compare them
```

Periphery runs its own build. Do not pass `-- -derivedDataPath ...` through to it: version
3.8 rejects that argument and prints xcodebuild's help.

**6. Sanitizers before a release:**
`swift test --package-path <package> --sanitize=thread`, then `--sanitize=address`.

## CI on macOS

- **The deployment target decides the runner.** An app targeting the newest macOS needs that
  SDK. GitHub publishes images per Xcode major (`runs-on: xcode-<N>`, arm64) alongside the
  per-OS images (`macos-<N>`), and a new Xcode major appears first as a public preview. A
  package that targets an older macOS can run `swift test` on the older image meanwhile.
- **A Darwin-only core needs macOS.** A package that imports Darwin APIs
  (`getattrlistbulk`, `fts`) cannot run its tests on Linux.
- **Per-PR job: optional.** macOS minutes cost about 10× Linux on private repositories. The
  desktop stack's default is no per-PR job: the release workflow runs the gate on the tag. Add
  a per-PR job when agents push, or when `main` must stay green between releases. When you do:
  - run the same `scripts/verify.sh`;
  - add a `paths` filter (`**.swift`, `**.pbxproj`, `.swift-format`, the script, the workflow);
  - add `concurrency: cancel-in-progress: true` and a `timeout-minutes`.

## Measured example: a Swift 6 macOS app (Pichay, October 2026)

Xcode 27.0, Swift 6.4, deployment target macOS 27. App target in Swift 6 mode with
`SWIFT_DEFAULT_ACTOR_ISOLATION = MainActor`; a nonisolated SwiftPM core with 34 tests.

| Measurement | Value |
| --- | --- |
| Compiler warnings before | 16, all `Text + Text` deprecated in macOS 26 |
| `swift format lint`, default config (2 spaces) | 6,245 findings, 5,398 of them `Indentation` |
| Same, 4 spaces / 140 columns, before reformatting | 701 findings |
| One `--in-place` run | 24 files, +698 / −440 lines |
| After reformat | 3 findings, all `TypeNamesShouldBeCapitalized` on template types |
| `scripts/verify.sh`, warm | about 10 s on an M-series Mac |
| Periphery, default flags | 9 findings, 4 of them false positives on a `Hashable` cache key |
| Periphery, with the config above | 4 real findings |

The `Text + Text` fix is mechanical. Interpolate styled `Text` values into one `Text`:

```swift
// Before (deprecated):
Text(label).fontWeight(.semibold) + Text(" \(suffix)")
// After:
Text("\(Text(label).fontWeight(.semibold)) \(suffix)")
```

## Related

- [ci-cd-pipeline-strategy.md](../conventions/ci-cd-pipeline-strategy.md) — the one-gate,
  tiered-checks model this page applies.
- [swift-modular-architecture.md](./swift-modular-architecture.md) — why logic lives in the
  package, which is what makes `swift test` the fast half of the gate.
- [swift-from-react-equivalences.md](./swift-from-react-equivalences.md) — the TypeScript
  analogues: swift-format is Prettier plus some ESLint, lefthook is the same tool.
- [stacks/desktop-swift.md](../stacks/desktop-swift.md) — the stack recipe that links this page.
