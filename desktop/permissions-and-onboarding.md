# Permissions & Onboarding

_How a desktop (Electron) app checks, requests and displays the OS media permissions behind
capture — microphone, screen, system audio, camera — per platform; the first-run onboarding flow;
and the traps that cost days the first time round._

> **Framing.** Generalized from two real products: a screen recorder (screen + mic + optional
> camera) and a meeting overlay (mic + system/call audio, no camera). Product names appear only
> as examples. Electron `44.x` / macOS 14–26 is the reference; the platform facts are dated
> where they matter.

---

## TL;DR — the checklist for the next app

1. **Enumerate the real OS gates first, per backend, not per feature.** "System audio" on macOS
   is one feature but *two possible backends with two different permissions* — see
   [Which backend answers "system audio"](#which-backend-answers-system-audio--and-why-it-decides-the-permission).
   Pick the backend whose permission you can actually check and request.
2. **Model only permissions you can check.** A status card that says "Granted" for two kinds
   while a third, invisible gate blocks capture is worse than no card. If a gate cannot be
   queried, either avoid the backend that needs it or show it as "verified by attempting", never
   as a fake row.
3. **Know who macOS thinks is asking.** From a terminal/IDE (`bun run dev`), TCC attributes every
   request to the **terminal**, not to Electron — see
   [The responsible process](#who-macos-thinks-is-asking-the-responsible-process). Give the
   packaged app a real `appId`/`productName`/icon so it is recognisable in System Settings.
4. **Status is re-read, never cached:** on mount, on window `focus`, and after every capture
   attempt. **The attempt is the truth** — a `NotAllowedError` from the real acquisition marks the
   channel denied whatever the pre-check said.
5. **Never claim to run with nothing captured.** If every wanted channel fails, stay stopped with
   the errors visible; Start is the retry.
6. **Prove it with a separate-process probe and a self-skipping E2E** — not with mocks alone. The
   probe reproduces the OS behaviour deterministically in ~3 s; the E2E guards the decision on
   every Electron bump.

---

## The macOS permissions behind capture

macOS gates media access through TCC ("Transparency, Consent, and Control"). Each gate is a
separate *service*, listed in **System Settings → Privacy & Security**, keyed on the app's
`CFBundleIdentifier` + code signature. What matters for an Electron app:

| Gate                              | TCC service                | System Settings pane                                     | Info.plist key needed to *prompt*  | Electron 44 can **check** it?                                        | Electron 44 can **request** it?                                                  |
| --------------------------------- | -------------------------- | -------------------------------------------------------- | ---------------------------------- | -------------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| Microphone                        | `kTCCServiceMicrophone`    | Microphone                                               | `NSMicrophoneUsageDescription`     | `systemPreferences.getMediaAccessStatus("microphone")`               | `askForMediaAccess("microphone")` — prompts only while `not-determined`          |
| Camera                            | `kTCCServiceCamera`        | Camera                                                   | `NSCameraUsageDescription`         | `getMediaAccessStatus("camera")`                                     | `askForMediaAccess("camera")`                                                    |
| Screen Recording                  | `kTCCServiceScreenCapture` | Screen & System Audio Recording (macOS 15+; "Screen Recording" before) | —                                  | `getMediaAccessStatus("screen")`                                     | **no prompt API** — a 1×1 `desktopCapturer.getSources()` nudges the OS dialog once, while `not-determined`; afterwards only System Settings |
| System Audio Recording (14.2+)    | `kTCCServiceAudioCapture`  | same pane, listed as "System Audio Recording Only" when alone | `NSAudioCaptureUsageDescription`   | **no** — `getMediaAccessStatus` accepts only `microphone`/`camera`/`screen` | **no** — only a real CoreAudio Tap capture prompts, and only if the *responsible process* carries the key |

Deep links (open with `shell.openExternal`):

```
x-apple.systempreferences:com.apple.preference.security?Privacy_Microphone
x-apple.systempreferences:com.apple.preference.security?Privacy_Camera
x-apple.systempreferences:com.apple.preference.security?Privacy_ScreenCapture   ← also the system-audio pane
ms-settings:privacy-microphone   (Windows)
ms-settings:privacy-webcam       (Windows)
```

`getMediaAccessStatus` returns `not-determined | granted | denied | restricted | unknown`. Map
`restricted` (parental controls/MDM) to your own `denied`; map anything you do not recognise to
`granted` rather than hard-blocking on a value a future OS added.

### Windows and Linux

- Windows gates microphone/camera through the global privacy settings (`getMediaAccessStatus`
  works); it **never gates screen capture** — `screen` reads `granted` everywhere. There is no
  `askForMediaAccess`; deep-link `ms-settings:` instead.
- Linux gates nothing through Electron's API — everything reads `granted`. That is not the same
  as "the feature exists": system-audio capture has **no documented Electron path on Linux**
  at all, so report the *channel* as `unavailable` (see the four-state model below), never
  the permission as denied.

---

## Which backend answers "system audio" — and why it decides the permission

The renderer asks for system audio with Chromium's legacy constraint form:

```ts
navigator.mediaDevices.getUserMedia({
  audio: { mandatory: { chromeMediaSource: "desktop" } },
  video: { mandatory: { chromeMediaSource: "desktop", chromeMediaSourceId, maxWidth: 2, maxHeight: 2, maxFrameRate: 1 } },
});
// then stop + remove the (unavoidable) video track
```

Which *backend* answers is a process-wide Chromium feature fixed at startup — and each backend
is gated by a different TCC service:

| Backend                                                   | Default in Electron | Gated by                                                | App can check/request it? | Notes                                                                                                                       |
| --------------------------------------------------------- | ------------------- | ------------------------------------------------------- | ------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| **CoreAudio Tap** (`MacCatapLoopbackAudioForScreenShare`) | yes, since `v39.0.0-beta.4` | System Audio Recording (`kTCCServiceAudioCapture`)      | **no**                    | Needs `NSAudioCaptureUsageDescription` on the responsible process to prompt at all; per-process audio exclusion possible. No fallback if tap creation fails. |
| **ScreenCaptureKit audio** ("Screen & System Audio Recording") | opt-in via `disable-features` | Screen Recording (`kTCCServiceScreenCapture`)           | yes                       | Covered by the permission the app already models; requires macOS 13+.                                                        |

**Decision rule.** If your app cannot present a third permission honestly (Electron 44 cannot
check or request it), opt out of the tap and keep system audio under Screen Recording:

```ts
// src/main/index.ts — before app.whenReady(); app.commandLine is inert afterwards.
// Never clobber an existing --disable-features value: appendSwitch replaces it.
const existing = app.commandLine.getSwitchValue("disable-features");
const features = process.platform === "darwin" ? ["MacCatapLoopbackAudioForScreenShare"] : [];
const merged = [...existing.split(",").filter(Boolean), ...features.filter((f) => !existing.includes(f))];
if (merged.length > 0) app.commandLine.appendSwitch("disable-features", merged.join(","));
```

Keep the decision as **data in a pure function** (`resolveDisabledChromiumFeatures(platform)`)
so it is unit-tested on its own, and pin it with the E2E below. Re-evaluate when Electron gains a
real check/request API for System Audio Recording, when Chromium removes the flag, or when the
product needs per-process audio exclusion (only the tap offers it).

**Symptom that tells you which gate you hit.** Both checkable kinds read `granted`, the screenshot
path (`desktopCapturer` thumbnails) works, yet system audio rejects with
`NotAllowedError: Permission denied by system` — that is the tap's permission, not Screen
Recording. Electron's docs describe the missing-key case as a "dead stream with no error"; on
macOS 26 / Chromium 152 it is a thrown `NotAllowedError`. Handle both.

---

## Who macOS thinks is asking: the responsible process

TCC attributes a request to the **responsible process** — the nearest ancestor that LaunchServices
started as an app. Consequences:

- **In development** (`bun run dev` → electron-vite → `Electron.app` as a child of your shell),
  the responsible process is the **terminal or IDE** (cmux, Cursor, Terminal, VS Code…). That is
  why "Electron" never shows up in System Settings, why the app reports whatever grants the
  terminal has, and why the terminal's `Info.plist` decides whether a prompt can appear at all
  (Electron's docs: _"If you are running Electron from another program like a terminal or IDE
  then that parent program must contain the Info.plist key"_). To test capture in dev: grant
  Microphone + Screen Recording **to the terminal**.
- **The packaged app** has its own identity. Replace scaffold placeholders before anyone tests
  permissions: `appId` (`CFBundleIdentifier`), `productName` (`CFBundleDisplayName`), a real
  `build/icon.icns`. Otherwise the privacy lists fill with unlabelled, generic-icon entries
  nobody can tell apart (unsigned/ad-hoc rebuilds can even create a fresh entry each time).
  Keep `appId` stable: it namespaces `userData` and the TCC rows.
- **Usage-description keys live in the packaged `Info.plist`** (`mac.extendInfo` in
  electron-builder). The stock `Electron.app` already ships the common ones, so a missing key
  is rarely the dev-mode cause — attribution is.
- **A Screen Recording change may need a full relaunch** of the process to take effect (a macOS
  limitation; microphone/camera usually update live). Hiding the window is not a relaunch — a
  tray-resident app needs an explicit Quit.

Cheap diagnostics: `ps -o pid,ppid,comm` up the ancestry to find the responsible app;
`plutil -p <App>.app/Contents/Info.plist | grep UsageDescription` for the keys;
`/usr/libexec/PlistBuddy -c "Print :CFBundleIdentifier" <built app>/Contents/Info.plist` to read
identity back from the real bundle, never from config.

---

## The split: thin OS wiring, pure decisions, one hook

- **Main, pure** (`permission-status.ts`) — `checkPermission(kind, { platform, getMediaAccessStatus })`.
  Takes the platform and the Electron call as parameters, so the mapping (platform gating,
  `restricted → denied`, unknown → granted, throw → denied on macOS only) is unit-tested with
  no Electron process. Four states, not a boolean:

  ```ts
  type PermissionState = "granted" | "denied" | "not-determined" | "unavailable";
  //  granted         — OS says yes, OR this platform never gates the kind
  //  denied          — OS says no (includes restricted)
  //  not-determined  — macOS only: never asked; the one state a native prompt can still appear in
  //  unavailable     — reserved for a kind this build genuinely cannot request
  ```

  A boolean collapses "the user said no" and "this platform never asks" into the same `false`,
  which is exactly the distinction onboarding exists to show.

- **Main, wiring** (`permissions.ts`) — `checkPermissions` / `requestPermission` /
  `openSystemSettings` over `systemPreferences`, `desktopCapturer`, `shell`. Deep-link panes are
  data (`settingsPaneFor(kind, platform)`), also pure.
- **IPC** — `permissions:check`, `permissions:request`, `permissions:open-settings`; validate
  `kind` against the closed set before any Electron call. See [IPC Contract](./ipc-contract.md).
- **Renderer** — one hook/context reads status; onboarding, the Settings card and the capture
  meter **share the same status→label mapping** so no two surfaces describe one OS state with
  different words.

### `requestPermission` — "Grant" always does something visible

1. Already `granted` → return the re-read status, touch nothing.
2. macOS + microphone/camera + `not-determined` → `askForMediaAccess` (native prompt).
3. macOS + screen + `not-determined` → 1×1 `desktopCapturer.getSources()` nudge; re-check.
4. Anything else (already denied, screen after the first time, Windows) → open the System
   Settings pane.
5. **Always return the re-read state**, never the assumed outcome of the attempt.

Hide "Grant" once the state is `granted`; keep "Open System Settings" (revoking happens there
too).

---

## Status is re-read, and the attempt is the truth

Electron/macOS expose **no "permission changed" event**. Three moments keep the UI honest:

1. **On mount** of any surface that shows status.
2. **On window `focus`** — the user left for System Settings and came back. Plain
   `window.addEventListener("focus", recheck)` in the renderer is enough (the DOM focus event
   mirrors `BrowserWindow` focus). Without it, a first-run overlay granting a permission on top of
   an already-mounted capture panel leaves the panel stale until the next Start.
3. **After every capture attempt** — a prompt may just have resolved, or a refusal is only known
   for real once acquisition was tried.

Then, per capture channel, compute one state from four inputs, the same way on every platform:

```ts
type CaptureChannelState = "unavailable" | "denied" | "available" | "active";
function computeCaptureChannelState(i: { supported: boolean; permission: PermissionState; requested: boolean; acquired: boolean }) {
  if (!i.supported) return "unavailable";          // no implementation on this platform — beats everything
  if (i.permission === "denied") return "denied";  // never "active" for a refused channel
  if (i.requested && i.acquired) return "active";  // a live track exists right now
  return "available";                              // nothing blocks it, nothing is captured
}
```

and feed `permission` from **both** sources: the OS pre-check *or* the last real attempt:

```ts
// NotAllowedError is the spec'd DOMException name for a refused getUserMedia/getDisplayMedia.
// Match by name: a DOMException is not `instanceof Error` in every realm (jsdom vs Chromium).
const isPermissionRefusal = (e: unknown) => typeof e === "object" && e !== null && (e as { name?: unknown }).name === "NotAllowedError";
permission: refusedByLastAttempt ? "denied" : preCheck ?? "not-determined";
```

Rules that fall out of this:

- **A refusal from the attempt wins over a "granted" pre-check** — the pre-check cannot see every
  gate (see the backend section). Re-test on the next Start; never remember a refusal across
  attempts.
- **Never "running" with nothing captured.** Two independent channels, each in its own
  try/catch, so one failing never masks the other — but if *every* wanted channel fails, tear
  down and stay stopped with the errors visible. Start is the retry; "Stop" would be a lie.
- **Distinct words for every non-active state** (`Off`, `Denied`, `Unavailable`) and a moving bar
  only for `active`, so a captured-but-silent channel never looks like an off one.
- Show the platform's own error text (`message`), never `Name: message`, never a fabricated one.

---

## Onboarding

First-run overlay: **welcome → one step per permission kind → done**.

- Each step fetches the **real** current status on entry and shows it — never inferred from a
  preference or from silence ("Checking…" until the IPC answers).
- "Grant" → `permissions:request`; display exactly what comes back, including a still-denied
  result. "Open System Settings" and "Skip for now" are always available: a denial is a
  legitimate outcome, not a dead end.
- Re-check the current step on window `focus` (the "Open System Settings" path has no in-app
  click to hang a re-check off).
- **Replay from Settings** without touching the persisted "completed" flag — only finishing sets
  it, and only the first time.
- If some kinds are optional (a recorder's camera), keep the "required to finish" rule in a pure
  renderer helper (`requiredPermissionsMet(status)`), never in the main process.

---

## How to prove it

**Unit** — every pure decision: state mapping, platform gating, deep-link data, the Chromium
feature list, channel-state computation, refusal detection. Mock `electron` only for the thin
wiring file.

**Separate-process probe** (throwaway, never committed — record the results in the project's
progress log). A tiny Electron app: main reports `getMediaAccessStatus` for each kind, fetches a
`desktopCapturer` source, and a hidden window runs the app's *exact* `getUserMedia` constraints,
reporting `{ ok, errorName, errorMessage }` or the live track plus RMS samples over ~1.5 s. Run it
with the project's pinned Electron binary. Differential runs (same identity, same grants, one
variable) settle causes in minutes — e.g. default backend vs
`--disable-features=MacCatapLoopbackAudioForScreenShare`, silence vs `say "…"` playing (RMS `0`
vs `0.20`). Play a sound during the run: a live-but-silent track is otherwise indistinguishable
from a dead one.

**Self-skipping E2E** (`@playwright/test` `_electron.launch` of the **built** app, throwaway
`--user-data-dir`): read `getMediaAccessStatus` via `app.evaluate`; if microphone or screen is
not `granted` for the responsible process, `test.skip` with the reason. Otherwise: skip
onboarding, Start capture, assert every wanted meter reads `active`, no per-channel error, no
`denied`, and — if you opted out of a backend — that `app.commandLine.getSwitchValue("disable-features")`
really contains it; then Stop. CI runners skip honestly; a developer machine with grants runs it
for real, and it is the regression check on every Electron bump.

**What only a human can do**: the packaged app's very first TCC prompts. Windows and dialogs
spawned from an automated process tree may never reach the interactive session — record it as
"not yet exercised" rather than faking a pass.

---

## Gotchas

- ⚠️ **Three macOS gates, not two, behind "mic + system audio"** — unless you opt out of the
  CoreAudio Tap backend. Decide this on day one; it shapes the whole permission model.
- ⚠️ **Dev permissions are the terminal's.** Do not chase "Electron" in System Settings; grant to
  the terminal, or test with the packaged app.
- ⚠️ **Screen recording has no prompt API** and once denied only System Settings helps — and the
  change may need a relaunch.
- ⚠️ **Without Screen Recording, `desktopCapturer.getSources()` still resolves** — with blank
  thumbnails or an empty list. Never read "no sources" as "nothing to capture".
- ⚠️ **`appendSwitch("disable-features")` replaces** an existing value; merge, don't clobber.
- ⚠️ **`DOMException` ≠ `instanceof Error` across realms**; match refusals by `name` and read
  `message` defensively — tests in jsdom will otherwise render `NotAllowedError: …`.
- ⚠️ **Scaffold placeholders** (`com.example.app`, `productName: Desktop`, the framework's
  default icon) reach System Settings verbatim. Fix them before the first permission test.
- ⚠️ Inspect the **bundle's** `Info.plist`, not the `node_modules/.bin` shim — the shim has none,
  the `.app` has the keys; drawing conclusions from the shim cost a wrong hypothesis once.

## Related

- [Main Process Architecture](./main-process-architecture.md) — where the `register*` handlers live
- [IPC Contract](./ipc-contract.md) — the `permissions:*` / `capture:*` channels
- [Media Pipeline](./media-pipeline.md) — what the permissions unlock
