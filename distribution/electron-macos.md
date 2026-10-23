# Electron / electron-builder — shipping a signed, self-updating macOS app

_The Electron-specific half of the macOS release: where the files live, what the feed must
contain, how the app updates itself, and the order of publication that keeps a half-failed
release harmless._

Drawn from an Electron desktop app (electron-builder + electron-updater, artifacts on Cloudflare
R2, releases cut by release-please) that shipped `v0.1.0` through `v0.9.0` with this pipeline. The
stage order and the two publication rules are in [README.md](./README.md); the icon and install
window are in [macos-app-icon-and-dmg.md](./macos-app-icon-and-dmg.md). This doc is only what
Electron adds on top. The Sparkle equivalent is [swift-xcode-macos.md](./swift-xcode-macos.md).

**What is and is not proven.** Signing, notarization, publication, the public feed and the manual
download are proven by real releases. The *in-app self-update* (an installed older build finding,
downloading and installing a newer one) has no validation record in the source repository: its
backlog marks it "pending a second release", and a later page still says the updater path "cannot
be exercised without a packaged build one version behind the feed". Treat the client pattern below
as the design that shipped, not as a field-tested one, until you have run the test plan at the end.

---

## Where each piece lives

| Concern | Tool | Setting |
| --- | --- | --- |
| Sign + notarize | electron-builder (`@electron/notarize`) | `mac.hardenedRuntime: true`, `mac.notarize: true`, entitlements plists |
| Packages | electron-builder | `mac.target`: `dmg` (the download) **and** `zip` (the update archive) |
| Feed file | electron-builder | writes `latest-mac.yml` next to the zip |
| Update client | `electron-updater` in the main process | reads the feed URL baked into the packaged app |
| Hosting | any static bucket with a public URL | see layout below |

Build with `--publish never` and publish yourself. electron-builder can upload, but a separate
publish job is what lets you control ordering, pruning and the "everything exists" check.

The zip is not optional: the macOS update mechanism under electron-updater (Squirrel.Mac) installs
from a zip, not from a DMG. A DMG-only build produces a download page and no update path.

### Generic provider

Point the app at a plain URL instead of a hosting-specific provider:

```yaml
# electron-builder.yml
publish:
  provider: generic
  url: https://updates.<domain>/updates/   # public custom domain bound to the bucket
```

electron-builder bakes this URL into the packaged app (`app-update.yml`), so it is frozen into
every shipped binary. Bind the domain to the bucket **before the first release**: an installed app
cannot be repointed later without an update, and the update is what it cannot fetch.

---

## Bucket layout

One public bucket, three prefixes with different rules:

```text
download/<version>/<app>-<arch>.dmg    immutable, one folder per release
download/latest/<app>-<arch>.dmg       stable alias, overwritten each release (website links here)
updates/                               the live feed
├── latest-mac.yml                     the feed file; names the newest version, size and sha512
├── <app>-<version>-...-mac.zip        update archive (one per arch)
└── <app>-<version>-...-mac.zip.blockmap   enables differential downloads
```

- `download/<version>/` never changes after upload. A bad build is fixed with a higher version.
- `download/latest/` gives the website a URL that never needs editing.
- `updates/` is the only prefix the installed apps read. `latest-mac.yml` has no version in its
  name, so it is overwritten in place; the zips and blockmaps carry the version.
- If you also ship Windows, its `latest.yml` and installer share `updates/`. Upload each platform
  from its own artifact with an explicit `--include` list, never a blanket copy, so one platform's
  feed cannot overwrite the other's.

### R2 specifics

The S3-compatible API of Cloudflare R2 needs its **own access key and secret** (created under R2
API tokens). A general Cloudflare API token does not authenticate against it, and the failure looks
like a plain access-denied. Use any S3 client with `--endpoint-url https://<account>.r2.cloudflarestorage.com`
and `AWS_DEFAULT_REGION=auto`.

---

## Release order

The workflow shape that worked is one job per concern, with a single publisher:

```text
verify (lint, types, unit) -> build-mac (build, E2E on the built app, sign, notarize)
                           -> build-win                                   -> publish
```

- The build jobs only build and upload `dist/` as workflow artifacts. Only `publish` touches the
  bucket or the GitHub Release, so two parallel jobs can never race on one tag.
- Run the E2E against the exact bundle that then gets signed, and only sign after it passes.
- `publish` starts by failing if any expected file is missing (both DMGs, the zip, the blockmap,
  `latest-mac.yml`). `aws s3 cp --include` exits 0 when nothing matches, so without this check a
  half-empty artifact directory publishes a half-written release.
- Derive the version from `package.json`, the same source electron-builder names the files from,
  not from the tag name. A tag/package mismatch then cannot point the feed at a file that does not
  exist.

Publication order the README requires, and where this pipeline stands against it:

| Step | Required | In the source pipeline |
| --- | --- | --- |
| Versioned objects (`download/<version>/`, update zip + blockmap) | first | zip and blockmap upload with the feed file, see gap below |
| `download/latest/` alias | second | uploaded **after** the feed |
| `updates/latest-mac.yml` | last | uploaded together with the zip |

### Known gap: the feed is not guaranteed to land last

The source pipeline uploads the zip, the blockmap and `latest-mac.yml` in one recursive
`aws s3 cp` with three `--include` patterns. A recursive copy transfers files in parallel with no
ordering guarantee, so `latest-mac.yml` can become visible before the zip it describes finishes
(or fails). An installed app that polls in that window sees a newer version, requests a zip that is
missing or partial, and reports an error until the next check. The window is short and it heals
itself, so it is a robustness gap, not a proven incident; nobody has observed it.

**Recommendation:** make the feed the final upload of the whole release.

```bash
# 1. immutable objects and the update archive
aws s3 cp artifacts/mac/ "s3://$BUCKET/updates/" --recursive \
  --exclude "*" --include "*-mac.zip" --include "*-mac.zip.blockmap" --endpoint-url "$ENDPOINT"
# 2. installers: download/<version>/ then download/latest/
# 3. only now, the feed, as its own command
aws s3 cp artifacts/mac/latest-mac.yml "s3://$BUCKET/updates/latest-mac.yml" --endpoint-url "$ENDPOINT"
```

Do the same for `latest.yml` on Windows. A failure at any earlier step then leaves every installed
user on the previous version, which is the safe state the README describes. Consider a short
`Cache-Control` on the feed file so a CDN does not keep serving the old one.

---

## Pruning old versions

Without pruning, `download/<version>/` and the versioned files in `updates/` grow by one release
forever; only `download/latest/` and the feed overwrite themselves. The source pipeline keeps the
last **3** versions (`KEEP_VERSIONS: 3`), after the new release is fully published.

Why 3 and not 1: the live feed only ever needs the newest version, so anything older is there for
**manual rollback and for users downloading a specific version**, not for auto-update. A handful is
enough for that.

Details that matter:

- List `download/`, drop the literal `latest`, sort with `sort -V` (semantic order, so `0.10.0`
  sorts after `0.9.0`), and delete all but the newest N.
- For each stale version, remove `download/<v>/` and the files in `updates/` whose names contain
  that version. Because `latest-mac.yml` carries no version, the version match can never delete
  the live feed.
- Run it last, and make it tolerant of an empty listing, so a pruning problem cannot fail a
  release that has already shipped.

---

## The updater client

Keep the logic in a **pure reducer** in shared code and make the Electron adapter dumb. The source
app did it this way because the logic used to live inside the main-process module, where the only
way to test it was to package the app, and four of the six electron-updater events were being
dropped without anyone noticing.

```text
shared/updater-state.ts     pure: (status, event) -> status; canStartCheck; shouldCheckOnFocus
main/updater/auto-updater.ts   electron-updater events -> reducer events -> broadcast to windows
```

Status is a small state machine: `idle`, `checking`, `up-to-date`, `available`, `downloading`,
`ready`, `error`. Rules worth copying:

1. **A downloaded build outranks everything.** Once `ready`, a later empty or failed check must not
   hide the waiting update.
2. **Progress only advances a download.** A stray progress event against a resting state is ignored.
3. **A finished check stamps its own time.** In-flight states carry no timestamp, so the UI cannot
   show a stale one.
4. **No check while checking, downloading or ready.** electron-updater is not re-entrant during a
   download, and a manual button is the first thing that makes a second call reachable.

Adapter behavior:

- `autoDownload = true` and `autoInstallOnAppQuit = true`: download silently, install when the
  user quits on their own.
- Do nothing when `!app.isPackaged`. electron-updater needs the `app-update.yml` that only a
  packaged build has. Give development a simulator that forces each status so the UI can be built
  without packaging.
- **Errors become a status, never a throw.** An update failure must not take the app down.
- **Fold in the rejection of `checkForUpdates()`.** It rejects on network failures and the
  `error` event does not always fire for them, so catch the promise and feed the same error event.
- **Broadcast to every window**, and expose the last status for windows that mount later. Sending
  to one window leaves the others showing stale state.
- Triggers, for an app that can stay open for days: on startup, every 6 hours, when a window
  regains focus (throttled; the source uses 30 minutes because a check can pull a large download),
  and a manual "Check for updates" button. Launch-only checks miss users who never quit.
- A "Restart" button calls `quitAndInstall()` only when the status is `ready`.

### Never interrupt live work

Installing means quitting. **Do not restart or install while the user is in the middle of
something that cannot be resumed** (a recording, a live call, an export, an unsaved edit). Show the
"update ready" state and let the user choose the moment, or defer to the next quit, which
`autoInstallOnAppQuit` already does. If an app is never quit, a visible banner is what keeps the
update from waiting forever.

---

## Signing requirement

Squirrel.Mac refuses to apply an update to or from an unsigned app, and the new version's signature
must match the installed one's. Consequences:

- A locally built unsigned or ad-hoc build **cannot** be used to test updating. You need a signed
  (and in practice notarized) older build installed from a real release artifact.
- Rotating the signing identity between releases breaks the update path for everyone already
  installed.

Version numbers matter just as much: the feed only triggers an update when it advertises a version
**higher** than the installed one. Let release-please (or your release tool) own `package.json`, and
build the tag from it, so the number baked into the app, the artifact names and the feed always
agree. See [monorepos/release-automation.md](../monorepos/release-automation.md).

---

## Test plan

Do this once before trusting the updater, and again after any change to signing, the feed or the
client.

1. Install the **previous signed release** from its published DMG, not a local build. Launch it
   once and leave it running.
2. Publish the next release through the real pipeline.
3. Fetch `updates/latest-mac.yml` and confirm it names the new version, and that the zip it lists
   returns 200.
4. In the running old build, press "Check for updates" (or bring a window to focus after the
   throttle, or wait for the timer). Watch the states: `checking`, `downloading` with progress,
   then `ready`.
5. Press Restart. Confirm the app relaunches as the new version and opens normally (Gatekeeper
   accepts it).
6. Repeat while a long-running action is in progress and confirm nothing restarts on its own.
7. Break it deliberately once: block the feed URL and confirm the app shows an `error` status and
   keeps running.

## Related

- [README.md](./README.md): the six-stage spine and the publication rules
- [swift-xcode-macos.md](./swift-xcode-macos.md): the Swift / Sparkle sibling
- [macos-app-icon-and-dmg.md](./macos-app-icon-and-dmg.md): icon and DMG layout
- [electron-icon-and-dmg.md](./electron-icon-and-dmg.md): the electron-builder `dmg:` block, tray and dev Dock icon
- [../stacks/desktop-electron.md](../stacks/desktop-electron.md): the Electron stack recipe
- [../monorepos/release-automation.md](../monorepos/release-automation.md): version drift and release-please
- [../monorepos/ci-cd-pipelines.md](../monorepos/ci-cd-pipelines.md): tag-driven pipelines
