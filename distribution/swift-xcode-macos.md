# Swift / Xcode — shipping a signed, self-updating macOS app

_The Xcode- and Sparkle-specific traps between "it builds in Xcode" and "a stranger can download
it, install it, and receive the next version" — each one as the symptom it actually presents._

Drawn from a native SwiftUI menu bar app taken from local build to five public releases: Developer
ID, notarization, a designed DMG, Sparkle 2 updates, artifacts on Cloudflare R2, releases cut by
release-please. Every entry below is a failure that actually happened, not a precaution.

The stage order and the publication rules are in [README.md](./README.md); the icon and install
window are in [macos-app-icon-and-dmg.md](./macos-app-icon-and-dmg.md). This doc is only what
Xcode and Sparkle add on top.

---

## Shape of the pipeline

Keep the release as **shell scripts the workflow calls**, not as workflow steps. The same scripts
then run on a laptop when something fails at 1 a.m., which is the only way to debug notarization.

```text
scripts/release/
├── common.sh                shared paths, team id, notarytool credential switch
├── archive-and-export.sh    xcodebuild archive + -exportArchive (Developer ID)
├── notarize-app.sh          submit, staple, spctl the .app
├── make-dmg.sh              stage, hdiutil, sign, notarize, staple the .dmg
├── make-update-zip.sh       ditto -c -k --keepParent → Sparkle archive
├── generate-appcast.sh      sign the feed with the EdDSA key
└── publish-r2.sh            immutable objects → latest alias → feed
```

The one piece of cleverness worth having is a credential switch, so the same script notarizes with
a keychain profile locally and an App Store Connect API key in CI:

```bash
notary_args() {
  if [[ -n "${APPLE_API_KEY_PATH:-}" ]]; then
    printf -- '--key %s --key-id %s --issuer %s' \
      "$APPLE_API_KEY_PATH" "$APPLE_API_KEY_ID" "$APPLE_API_ISSUER"
  else
    printf -- '--keychain-profile %s' "$NOTARY_PROFILE"
  fi
}
```

---

## Xcode traps

### The project file format outruns the CI runner

**Symptom:** the runner fails immediately with *"future Xcode project file format"*.

A current Xcode writes a newer `objectVersion` into `project.pbxproj` the first time it touches
the file, and a GitHub `macos-*` runner image is typically one major version behind. Xcode 27's
format (`objectVersion = 110`) cannot be read by the Xcode 26 on a `macos-26` runner.

**Fix:** lower the project format to the newest one *both* versions read — 77 at the time of
writing — and verify nothing you use needs the newer one (filesystem-synchronized groups and SPM
references both predate it). Then pin the toolchain explicitly and fail loudly rather than
building with whatever `xcode-select` points at:

```yaml
- name: Pin Xcode
  run: |
    XCODE="$(ls -d /Applications/Xcode_26*.app 2>/dev/null | sort -V | tail -1)"
    if [ -z "$XCODE" ]; then
      echo "No Xcode 26 on this runner; available:" >&2
      ls /Applications | grep Xcode >&2
      exit 1
    fi
    sudo xcode-select -s "$XCODE"
```

### Xcode only generates the Info.plist keys it knows

**Symptom:** a key you set is simply absent from the built bundle, with no warning.

With `GENERATE_INFOPLIST_FILE = YES`, Xcode synthesizes the plist from `INFOPLIST_KEY_*` build
settings — and it only understands the keys it ships a setting for. Anything else
(`LSMultipleInstancesProhibited`, Sparkle's `SUFeedURL` / `SUPublicEDKey`) is dropped silently.

**Fix:** add a real `Config/Info.plist` holding *only* the keys Xcode will not generate, and point
`INFOPLIST_FILE` at it. The generated keys and the file are merged, so the two mechanisms coexist.
Verify in the built product, never in the source file.

### Entitlements: build settings until suddenly they are not

**Symptom:** everything works in development; in the signed build, a specific capability fails
with a vague message from whatever framework needed it. In this project: Sparkle's *"an error
occurred while running the updater"* — checking and downloading an update worked, installing never
did.

Xcode generates an entitlements file from build settings (`ENABLE_APP_SANDBOX`,
`ENABLE_USER_SELECTED_FILES`, …) as long as you only need entitlements it has settings for. A
sandboxed app that needs to reach a helper's XPC services has no such setting, so the entitlement
was never in the binary.

**Fix:** switch to a checked-in `.entitlements` file, restate the entitlements the settings used
to generate, and add the custom ones:

```xml
<key>com.apple.security.app-sandbox</key>
<true/>
<key>com.apple.security.files.user-selected.read-only</key>
<true/>
<!-- Sparkle sandboxing: without these, every install fails. -->
<key>com.apple.security.temporary-exception.mach-lookup.global-name</key>
<array>
  <string>$(PRODUCT_BUNDLE_IDENTIFIER)-spks</string>
  <string>$(PRODUCT_BUNDLE_IDENTIFIER)-spki</string>
</array>
```

Check the shipped artifact, not the project: `codesign -d --entitlements - <App>.app`.

**The consequence worth planning for:** versions released *before* the fix cannot self-update —
they will download and then fail forever. Those users need one manual reinstall from the DMG. An
updater bug is the one bug you cannot ship a fix for.

### The version must come from the tag, not the project

Leave `MARKETING_VERSION` and `CURRENT_PROJECT_VERSION` alone in the project and override them at
archive time, so a released build can never carry a stale number:

```bash
xcodebuild archive … \
  CODE_SIGN_STYLE=Manual \
  ENABLE_HARDENED_RUNTIME=YES \
  CODE_SIGN_IDENTITY="Developer ID Application" \
  DEVELOPMENT_TEAM="$TEAM_ID" \
  MARKETING_VERSION="$MARKETING_VERSION" \
  CURRENT_PROJECT_VERSION="$BUILD_NUMBER"
```

`MARKETING_VERSION` comes from the tag (`v0.4.0` → `0.4.0`). `CFBundleVersion` must be **strictly
increasing across releases** because that is what Sparkle orders updates by — deriving it from the
CI run number with an offset that clears local dev builds (`100 + github.run_number`) is enough,
and needs no state.

Keep the overrides conditional so a local run still uses the project's own values:

```bash
OVERRIDES=()
[[ -n "${MARKETING_VERSION:-}" ]] && OVERRIDES+=("MARKETING_VERSION=$MARKETING_VERSION")
…
xcodebuild archive … ${OVERRIDES[@]+"${OVERRIDES[@]}"}
```

### The test host is the app

**Symptom:** `xcodebuild test` or pressing Run fails with LaunchServices **error 20**, which says
nothing about duplicate instances.

Unit tests for an app target are hosted *by the app*. If the app is already running — and a menu
bar app is always already running — and the bundle declares `LSMultipleInstancesProhibited`, the
second copy cannot launch. Quit the app before testing; put that sentence in the README, because
the error sends everyone hunting in the wrong place.

The same sandbox that runs the app runs the tests: anything a test writes must go **inside the
container** (`~/Library/Containers/<bundle-id>/Data/…`), or it fails as a signal kill rather than
a readable error.

---

## Sparkle traps

### `generate_appcast` is inside DerivedData

Sparkle ships as an SPM binary artifact, so its tools are not on `PATH` and not in the repo. Find
the one that belongs to the resolved package:

```bash
GENERATE_APPCAST="$(find "$HOME/Library/Developer/Xcode/DerivedData" \
  -path "*artifacts/sparkle/Sparkle/bin/generate_appcast" -type f 2>/dev/null | head -1)"
[[ -n "$GENERATE_APPCAST" ]] || { echo "resolve packages first" >&2; exit 1; }
```

`generate_keys -x <file>` lives in the same directory and is how the EdDSA private key is
exported. **Losing that key strands every existing install**; leaking it lets someone else sign
updates for your users. It is the one secret with no recovery path — back it up offline.

### The appcast is regenerated, so it must start from the live one

`generate_appcast` writes the feed from the archives in a directory. Run it on a clean CI checkout
and you produce a feed containing only the version you just built, silently deleting the history.

**Fix:** download the currently-served feed into the working directory first. Entries whose
archives are absent are preserved.

```bash
curl -fsSL "$DOWNLOAD_BASE_URL/updates/appcast.xml" \
  -o dist/updates/appcast.xml || echo "No existing appcast; starting fresh"
```

### The feed URL is baked in, permanently

`SUFeedURL` is compiled into each build. An installed app checks the URL *it* was built with
forever, so changing the download host later orphans everyone already installed. Treat the public
base URL as permanent from the first public build, and inject it at archive time so Debug builds
can point at a test feed instead:

```
APP_FEED_URL=$DOWNLOAD_BASE_URL/updates/appcast.xml   # one build setting, one Info.plist key
```

### Restarting for an update ends whatever the app was doing

For a background/menu-bar utility this is a product decision, not a technical one: warn before
installing that the running session ends, allow postponement, and never silently resume after
relaunch. Decide it before the first release — it is in the update dialog, which you cannot patch
without shipping an update.

---

## Release plumbing traps

### A tag pushed by `GITHUB_TOKEN` triggers nothing

When release-please cuts the tag with the default token, GitHub deliberately does not fire
workflows from it — so the release workflow never runs and the tag sits there. Give release-please
a fine-grained **PAT** (Contents + Pull requests, read/write) instead.

### release-please already created the release

**Symptom:** *"a release with the same tag name already exists"*.

The deploy workflow's `gh release create` collides with the release release-please made when it cut
the tag. Upload to the existing one, and keep `create` only as the fallback for a hand-pushed tag:

```bash
gh release upload "$GITHUB_REF_NAME" "dist/<App>-$VERSION.dmg" --clobber ||
  gh release create "$GITHUB_REF_NAME" "dist/<App>-$VERSION.dmg" \
    --title "<App> $VERSION" --generate-notes
```

### Publish in the safe order, and refuse to overwrite

```bash
refuse_overwrite() {
  if s3 head-object --bucket "$R2_BUCKET" --key "$1" >/dev/null 2>&1; then
    echo "Refusing to overwrite existing object: $1" >&2
    exit 1
  fi
}
```

Then: immutable `download/<App>-<version>.dmg` and `updates/<App>-<version>.zip`
(`max-age=31536000, immutable`) → stable `download/latest/<App>.dmg` (`max-age=0`) → live
`updates/appcast.xml` (`max-age=0`) last. Rerunning a half-finished release then finishes
promotion instead of corrupting it.

### Give `workflow_dispatch` a rehearsal mode

Gate every publishing step on `if: github.event_name == 'push'`. A manual run then exercises the
entire build/sign/notarize/package path — the expensive, breakable part — while being structurally
incapable of touching production objects or the live feed.

### Serialize releases

```yaml
concurrency:
  group: release-macos
  cancel-in-progress: false
```

Cancelling a release between the artifact upload and the feed upload is exactly the state you
spent the whole publication order avoiding.

### Clean up credentials in an always-run step

A temporary keychain and materialized `.p8`/EdDSA key files must be removed with `if: always()`,
including on failure.

---

## Checklist

- [ ] Project format readable by the runner's Xcode; toolchain pinned with a loud failure
- [ ] Non-generatable Info.plist keys in a real `Config/Info.plist`, verified in the built bundle
- [ ] Checked-in `.entitlements`, verified with `codesign -d --entitlements -`
- [ ] Version from the tag, `CFBundleVersion` strictly increasing
- [ ] Hardened Runtime on, manual signing, Developer ID export
- [ ] App notarized and stapled; DMG signed, notarized, stapled, `spctl`-asserted
- [ ] Sparkle EdDSA private key backed up offline
- [ ] Appcast regenerated from the live feed, not from an empty directory
- [ ] Feed URL treated as permanent; Debug builds on a separate test feed
- [ ] release-please PAT wired; release uploaded, not created
- [ ] Publication ordered, overwrite-refusing, serialized, rehearsable, credential-cleaning
