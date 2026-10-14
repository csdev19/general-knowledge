# Electron on macOS — floating windows over full-screen apps without breaking the menu bar

_A tray popover, a recording control bar or a camera bubble has to appear over whatever the user
is looking at, including an app in full screen. The obvious Electron call for that also changes
the whole app, not just the window: kept on all the time, it blanks the app's own menu bar. This
page is the rule, why each piece is needed, and how the failure was isolated._

Drawn from an Electron screen-capture app with a menu-bar popover that is created ahead of the
first click and reused. The behaviour below was checked by hand on a dev build, against the same
build on `main`; it cannot be unit tested.

---

## The rule

For a floating window that must open over full-screen Spaces on macOS:

1. Create it as a **panel**: `type: "panel"`, plus `fullscreenable: false` and an always-on-top
   level (`setAlwaysOnTop(true, "screen-saver")`).
2. Join every Space, full-screen ones included, **only while the window is open**:

   ```ts
   private setSpaceBehavior(open: boolean): void {
     if (process.platform !== "darwin" || !this.window || this.window.isDestroyed()) return;
     this.window.setVisibleOnAllWorkspaces(open, {
       visibleOnFullScreen: open,
       skipTransformProcessType: true,
     });
   }
   // show(): position, setSpaceBehavior(true), show, focus
   // hide(): hide, setSpaceBehavior(false)
   ```

3. Always pass `skipTransformProcessType: true` in an app that has a Dock icon and a menu bar.

## Why each piece

**A plain window cannot open over a full-screen Space.** macOS moves it to a Space of its own,
which the user sees as a sudden switch to a black desktop showing only the popover. If the
window is reused rather than recreated (prewarmed so the first click is instant), it stays bound
to that Space: every later open lands there until the app restarts.

**`setVisibleOnAllWorkspaces(true, { visibleOnFullScreen: true })` alone changes the app.** By
default Electron transforms the whole process to `UIElementApplication` to get there: the
documented trade-off is that it "will hide the window and dock for a short time every time it is
called". A `UIElementApplication` has no Dock icon and no menu bar of its own.
`skipTransformProcessType: true` avoids the transformation.

**`type: "panel"` is the supported way to float over full-screen apps.** Electron's
`BaseWindowConstructorOptions` documents that it adds `NSWindowStyleMaskNonactivatingPanel` at
runtime so the window floats on top of full-screened apps, and that it appears on all Spaces.

**Full-screen Space membership must not stay on while the window is hidden.** With either
approach active from creation, the app's own full-screen window loses its menu bar: hovering the
top edge drops the window's title bar, but the macOS menu bar above it stays black and empty —
no menus, no clock, no status icons. Other apps in full screen are unaffected. Turning the
membership on in `show()` and off in `hide()` keeps the popover working over full screen and
leaves the app identical to before while the popover is closed.

## Diagnosis: A/B against the last good build

The black menu bar looked like a separate bug (a video player's letterboxing and a recording that
contains a menu bar both muddied it). What settled it was one swap, not more reasoning:

1. Keep everything else on the branch; put the window file back to its version on `main`.
2. Restart and repeat the exact gesture (full screen → hover the top edge).
3. Menu bar fine on `main`, black on the branch → the change caused it. Then change one thing at
   a time and repeat.

Keep the candidate version in a temporary file during the swap so it can be restored.

A second check came for free: the installed production build and the dev build run side by side.
Full-screening the dev build and opening each one's popover showed the old bug (production
jumps to the black Space) and the fix (dev opens in place) in the same setup.

## Tests worth keeping

Space behaviour itself needs a person and a screen. What a mocked `BrowserWindow` can pin, with
`process.platform` forced to `darwin` (CI may run on Linux):

- created with `type: "panel"`, `fullscreenable: false`, the always-on-top level;
- `setVisibleOnAllWorkspaces` not called at creation;
- `show` calls it with `true` and `skipTransformProcessType: true`; `hide` calls it with `false`.

The second assertion is the one that would have caught the regression.

## Measured example — Kaipu (2026-10-14)

Kaipu's tray capture panel (`capture-panel-window.ts`) is prewarmed and reused. Opened from the
menu bar over a full-screen video, it jumped to a black Space and stayed there. The first fix
(Space membership from creation, as the recording control bar already did) opened the panel
correctly but blanked the menu bar over a full-screen Kaipu window. `type: "panel"` with the
membership still on from creation did the same. `main`'s panel showed the menu bar normally,
which isolated the cause; the open-only toggle fixed both. Fix: kaipu-record PR #291
(NIW2-380).

The recording control bar and camera bubble still set the membership from creation, without
`skipTransformProcessType`. They only exist during a recording, so the effect is limited to that
time; they are candidates for the same treatment.

## Related

- [main-process-architecture.md](./main-process-architecture.md) — the window and tray lifecycle
  this popover lives in.
- [swift-macos-app-lifecycle.md](./swift-macos-app-lifecycle.md) — the native Swift side of
  non-activating panels: what they do to the app's own input.
- Electron, [BaseWindowConstructorOptions](https://www.electronjs.org/docs/latest/api/structures/base-window-options)
  (`type`: `panel`) and
  [`win.setVisibleOnAllWorkspaces`](https://www.electronjs.org/docs/latest/api/browser-window#winsetvisibleonallworkspacesvisible-options-macos-linux)
  (`visibleOnFullScreen`, `skipTransformProcessType`).
