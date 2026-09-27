# Swift / macOS — the AppKit lifecycle under SwiftUI, and how to see input failures that make no noise

_A SwiftUI `App` is created before `NSApplication` has finished launching. AppKit side effects
taken in that window can leave the app unable to receive its own mouse events — with no error,
no log line and no crash. This page is the rule, the way to diagnose it, and the facts about
macOS input that the diagnosis settled._

Drawn from a native SwiftUI menu bar app with borderless, non-activating panels on the desktop.
Everything below was measured with synthetic input on a probe build, not inferred from
documentation; the numbers are in the measured example at the end.

---

## The rule: the composition root builds, the app delegate acts

SwiftUI creates the `App` struct — and evaluates its stored properties — **before**
`NSApplication` finishes launching. A composition root held in one of those properties
(`private let environment = AppEnvironment.shared`) therefore runs its initializer at the earliest
possible moment of the process, earlier than any AppKit delegate callback.

That is fine for building objects. It is not fine for **side effects on AppKit**:

| Side effect taken in the `App` initializer | What happens |
| --- | --- |
| `NSEvent.addGlobalMonitorForEvents(...)` | the app's own windows stop receiving mouse events — measured, see below |
| `NSStatusItem` creation and `statusItem.button` access | the button is not available yet; documented across menu bar app write-ups |
| `NSApp.activate`, `setActivationPolicy` | activation state is not established; behaviour differs from a later call |
| `NSColorPanel.shared` and similar shared panels | a "update while view is being updated" crash is reported |

So split the two responsibilities:

```swift
@main
struct MainApp: App {
    @NSApplicationDelegateAdaptor(AppDelegate.self) private var appDelegate
    private let environment = AppEnvironment.shared   // builds and wires; no AppKit calls
    ...
}

final class AppDelegate: NSObject, NSApplicationDelegate {
    func applicationDidFinishLaunching(_ notification: Notification) {
        AppEnvironment.shared.applicationDidFinishLaunching()   // monitors, activation, status items
    }
}
```

- The **composition root** (`AppEnvironment`, `<app>Environment`, whatever it is called) only
  constructs adapters and use cases. Its `init` has no AppKit calls.
- **Lifecycle effects** — event monitors, activation, windows ordered front at launch, status
  items — start from `applicationDidFinishLaunching`, through one method on the root.
- A component that owns such an effect **guards itself** as well: `start()` installs immediately
  when `NSApplication.shared.isRunning`, otherwise waits for `didFinishLaunchingNotification`.
  The rule then survives a future caller who forgets it.

This is the same discipline the hub already asks of every stack — the app only wires
([architecture/](../architecture/README.md)) — applied to AppKit's clock: *wiring* is allowed
before launch, *acting* is not.

---

## Why it stays hidden, and the diagnosis that finds it

The failure has no signal. The app runs, timers fire, windows draw and move, the status item
opens its overlay; only mouse events to the app's own windows vanish. Console logs show the usual
AppIntents and activation noise and nothing about input. Reverting the feature fixes it, which
points at the whole feature rather than at one line.

Two things make it findable, and both are worth keeping as tools.

### A probe build under a separate bundle id

`LSMultipleInstancesProhibited` (set by most menu bar apps) is keyed by bundle id. Building the
same source with `PRODUCT_BUNDLE_IDENTIFIER=<id>.probe` gives a second app that runs **next to**
the one under test in Xcode, with its own sandbox container and its own log subsystem:

```bash
xcodebuild -scheme <App> -derivedDataPath build/probe-dd PRODUCT_BUNDLE_IDENTIFIER=<id>.probe build
"build/probe-dd/Build/Products/Debug/<App>.app/Contents/MacOS/<App>" &   # launch by pid, not `open`
```

Launching the binary directly (not `open`) gives a pid to find its windows by
(`kCGWindowOwnerPID` in `CGWindowListCopyWindowInfo`) and to kill without touching the other
instance. Do not override `PRODUCT_NAME` on the command line: it applies to every target in the
build and fails with "Multiple commands produce".

Without the separate id, the two instances fight: Xcode's ▶︎ reports *Finished running* at once
because the other copy already holds the id, and an instance paused under the debugger cannot be
killed from a shell (`kill -9` leaves it in state `SX`).

### One real click, logged at three stages

Post a genuine click through the HID tap — `CGEvent(mouseEventSource:mouseType:...)` with
`.post(tap: .cghidEventTap)`; the terminal needs Accessibility trust — on the center of one of
the app's windows, and log at three points with `os.Logger` under the probe's subsystem:

1. an app-level `NSEvent.addLocalMonitorForEvents(matching: .leftMouseDown)` — did
   `NSApplication` dispatch it at all;
2. the window's `sendEvent(_:)` override — did the window get it;
3. the view's `mouseDown` / the handler — did the code run.

Read with `/usr/bin/log show --last 30s --predicate 'subsystem == "<id>.probe"'`. Use the full
path: in zsh, `log` is a shell builtin and prints nothing.

The stage that is missing names the layer. When even stage 1 is missing while the app's window
is under the cursor, WindowServer routed the event to the app and AppKit dropped it before
dispatch — the signature of this failure. Then bisect by variant, one change per build.

Keep the probe as a script in the repository (`scripts/click-smoke.sh`: build under the smoke
id, launch, click one window, grep the log, exit 1 on silence) and run it before merging anything
that touches windows, events or launch. It is thirty seconds, and it is the only test that
catches this class.

---

## Facts about macOS input this settled

- **A borderless, non-activating `NSPanel` receives clicks.** With `.borderless` and
  `.nonactivatingPanel`, `canBecomeKey` false and `acceptsFirstMouse` true, a click on the
  panel reaches its view without activating the app — and it does **not** pass through to the
  window underneath. Any design that assumes pass-through ("the click goes to the app below, so
  a global monitor will see it") is wrong: WindowServer delivers it to the panel's process, and a
  global monitor never sees it.
- **Global and local monitors are disjoint.** A global monitor observes events dispatched to
  *other* applications and can neither modify nor block them; a local monitor observes the app's
  own. Double clicks on your own window need the window's `mouseDown` (`clickCount == 2`);
  double clicks elsewhere need the global monitor. Both, if the feature wants both.
- **Mouse monitors need no permission; key monitors do.** Apple's guide: a global monitor "may
  only monitor key events if accessibility is enabled". Mouse-down monitoring worked in a
  sandboxed app with no prompt. Apple DTS adds that permission state is cached on a development
  Mac and can mislead — verify sandbox behaviour on a Mac that has never seen the app.
- **Borderless-window input is a moving target on recent macOS.** Apple acknowledged a
  regression on a 26.3 release candidate where clicks were not delivered to borderless windows at
  all. Measure on the OS you ship for; do not assume a behaviour carried over.
- **Timing of `addGlobalMonitorForEvents` is undocumented.** Apple's guide covers when to
  *remove* a monitor ("much earlier in the application life cycle" than `dealloc`) and nothing
  about when to add one. No forum or Q&A thread describes the failure above; the general rule
  ("AppKit setup in `applicationDidFinishLaunching`, not in the `App` initializer") is widely
  stated, its consequence for event delivery is not.

---

## The debugging rule that would have saved a day

Two fixes were applied before any measurement — an explicit `ignoresMouseEvents = false` and a
domain change — on the theory that the panel was letting clicks through. Both were wrong, and
each rebuild reset the user's manual test. The measurement, once done, took three builds and
settled the cause in under an hour.

**Reproduce with instrumentation before changing code.** When the symptom is "nothing responds",
the first artifact is a log that shows where the event stops, not a fix. Systematic debugging
already says this; this page is the AppKit-specific reason it matters — the failure mode is
silent, so intuition has nothing to work from.

---

## Measured example — a desktop-pets menu bar app (MyPets, 2026-09-27)

The app (`LSUIElement`, sandboxed, one 96×96 pt non-activating panel per pet) gained a
"double-click scares pets" feature: a global mouse-down monitor installed from the composition
root's initializer. From then on single clicks on a pet no longer triggered the hop, and the menu
bar overlay's switches stopped responding. Reverting the feature restored both.

Probe build (`dev.niway.my-pets.probe`, one seeded pet, logging at the three stages), identical
synthetic clicks on the pet's center, four variants:

| Variant | Stage 1 (app monitor) | Stage 2 (`sendEvent`) | Stage 3 (handler) |
| --- | --- | --- | --- |
| Stable baseline, no global monitor | yes | yes | yes |
| Monitor installed in `AppEnvironment.init` | **no** | no | no |
| Same build, `start()` never called | yes | yes | yes |
| Same build, `start()` on `didFinishLaunching` | yes | yes | yes |

Fix: the monitor's `start()` defers until the app is running; lifecycle effects moved to
`AppDelegate.applicationDidFinishLaunching`; the on-window double-click path restored (clicks on
a pet reach its panel, see the facts above). Verified end to end on the fixed build: single click
→ hop, double click on a pet → hop then flee, double click 100 pt beside it in another app → flee
through the global monitor. The click smoke script now lives in the repository and passes.

---

## Related

- [../stacks/desktop-swift.md](../stacks/desktop-swift.md) — the assembly recipe this page belongs to.
- [../distribution/swift-xcode-macos.md](../distribution/swift-xcode-macos.md) — the Xcode and Sparkle traps on the way to a signed, self-updating release.
- [../architecture/README.md](../architecture/README.md) — the app only wires; here applied to AppKit's launch timeline.
- Apple, [Monitoring Events](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/EventOverview/MonitoringEvents/MonitoringEvents.html) — global vs local scope, key events need accessibility, when to remove a monitor.
- Apple, [NSApplicationDelegateAdaptor](https://developer.apple.com/documentation/swiftui/nsapplicationdelegateadaptor).
- Apple Developer Forums, [global monitors in a sandboxed app](https://developer.apple.com/forums/thread/811443) (DTS: cached permission state; CGEventTap + Input Monitoring is the supported path for keys) and [borderless window interactions broken on a 26.3 RC](https://developer.apple.com/forums/thread/814875).
