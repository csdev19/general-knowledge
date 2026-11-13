# Native desktop — Swift, Tauri, others

**Status:** unprobed.

| Stack | Unattended driver | Fallback |
|---|---|---|
| Swift (macOS) | XCUITest, `XCTAttachment` screenshots | `screencapture -l <windowid> <file>` |
| Tauri | `tauri-driver` (WebDriver) on Linux and Windows; macOS unsupported as believed, to be probed | `screencapture` |
| Rust GUI (egui, iced, Slint) | none known | the app's own frame dump if it has one; `screencapture` |

`screencapture` needs the Screen Recording permission for the calling terminal, so it is a
**manual** capture: the evidence row says so and the run is not queueable. Write the first probe of
each stack back to this page.
