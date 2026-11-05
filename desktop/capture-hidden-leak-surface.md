# Capture-hidden windows — what the OS still draws

_Content protection (`setContentProtection(true)` in Electron; `NSWindowSharingNone` on macOS,
`WDA_EXCLUDEFROMCAPTURE` on Windows) removes a window's **own pixels** from a screen share or
recording. It hides pixels, not presence: everything the OS draws *about* the window stays in the
capture. This page is the checklist of those surfaces, what to do about each, and how to verify,
so an app that promises "hidden from screen share" does not learn about a leak from a recording._

Drawn from a desktop overlay (a memory assistant for live calls) that shipped "hidden" for weeks,
then found its main window's shadow in a recording as a shadowed empty rectangle, and five more
surfaces in the audit that followed. Verified by recording the display with a screen recorder
while the app was hidden; none of it can be unit tested, only encoded as pure decisions.

---

## The rule

A window is hidden only when **every layer below** is handled, and the status the app shows the
user is derived from what was actually applied, never from what was requested. A false "hidden"
is the one answer that can hurt.

| Layer | What leaks | What to do |
| --- | --- | --- |
| **Around the window** | The OS window shadow: the window server draws it as its own layer around the excluded pixels. A translucent material (`vibrancy`, acrylic) samples what is behind the window and may leak as a faint rectangle | `setHasShadow(false)` while hidden, back on while visible (macOS-only effect; unverified elsewhere). If a faint rectangle remains, switch to a solid fill while hidden |
| **Menu bar and Dock** | The menu bar shows the **active app's name**; a full-display share captures the menu bar. The tray/status item, the Dock icon and the Cmd+Tab entry | Make every window a person clicks during a share a **non-activating panel** (`type: "panel"` on macOS) shown with `showInactive()`. The base window's bundle name is the residual: keep it short. Hide the tray and the Dock entry while hidden (`Tray` has no hide: destroy and recreate; `app.dock.hide()`) |
| **System popups the app did not draw** | The macOS press-and-hold accent popover, the IME candidate window, native dialogs and sheets (`dialog.show*`), notifications, native context menus, tooltips (`title`), `<select>`, `<datalist>`, date and colour pickers | Accent popover: `ApplePressAndHoldEnabled=false` in the app's own defaults domain (a held key repeats; dead keys still work). IME: cannot be suppressed; document it. Dialogs: refuse while hidden, with product copy. Everything the renderer owns: draw it in-DOM or in a protected popover window; a lint rule against `title=` is cheap |
| **The pointer** | The share captures the cursor, not the window: a hand, I-beam, resize cursor or drag ghost over an invisible window gives it away | Force the arrow over every window while hidden, lock resizing, cancel `dragstart`; a lint rule against `cursor … !important` keeps the base rule winning |
| **Platform gates** | Windows before build 19041 (10 2004) has no `WDA_EXCLUDEFROMCAPTURE`: the window captures as a **black rectangle**. Linux: compositor-dependent, no guarantee | Report "cannot hide" (status says visible) from a pure gate on `os.release()`; never claim hidden where the platform cannot deliver |

## Why each piece

- **Shadow and material are the window server's, not the window's.** Exclusion is applied to the
  window's content layer; the shadow is composited separately and the material backdrop is sampled
  from the desktop. Nothing in the renderer can affect them.
- **Activation is app-wide.** Any click on a normal window activates the app, and the menu bar
  follows the active app. `showInactive()` only avoids activation at show time; the click that
  follows still activates unless the window is a non-activating panel.
- **A system popup is a window of its own**, without the app's protection, positioned over the
  hidden one: it is as visible as any other app's window.
- **The status must be derived.** Compute `{ visible, canHide }` from each window's live
  protection plus the platform gate; a build or platform that cannot protect reads "visible"
  whatever was requested. Indicators are status only; the mode changes in one place.

## Verify by recording, in every state

Record the display with a screen recorder (not the app's own capture) while hidden, and look at
the file, in each state the product can be in — not only the overlay during a call:

1. Base window open, over text (shadow, faint rectangle).
2. Secondary panel open; a menu or popover open (shadow; the menu bar name).
3. Base window focused (menu bar name; Dock and Cmd+Tab if the Dock is on screen).
4. Hidden mode **before** a session starts (tray, Dock).
5. Typing: hold a letter key; type through an IME.
6. Any action that opens a native dialog (export, save, open).
7. The pointer over every window, text selection dragged, a resize edge hovered.
8. Each platform's floor (Windows build, Linux).

Keep the list as a release gate with a recording linked per row; a probe that covers one state
proves only that state.

## Related

- [floating-windows-over-full-screen.md](./floating-windows-over-full-screen.md) — `type: "panel"`
  and Space membership: the same non-activating panel, used for a different reason.
- [permissions-and-onboarding.md](./permissions-and-onboarding.md) — Screen Recording permission
  on macOS, which the app's own capture needs and which is separate from being captured.
