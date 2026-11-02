# Mobile — Expo, iOS simulator, Android emulator

**Status:** partial (Expo launch only, monorepo-template#30; deep links unconfirmed, Maestro not installed).

| Need | Driver |
|---|---|
| Screens the owner watches live, in T3 Code | T3 device tools (`device_open` returns an `agent-device` command; `device_screenshot`) |
| Scripted flows, Expo / React Native | Maestro: `takeScreenshot`, `startRecording` / `stopRecording` |
| Raw iOS simulator | `xcrun simctl io booted screenshot <file>`, `recordVideo` |
| Raw Android emulator | `adb exec-out screencap -p > <file>`, `screenrecord` |

In T3 Code, drive devices through its device tools, not `simctl` or `adb` directly. A deep link to
the route under test beats tapping through the app.
