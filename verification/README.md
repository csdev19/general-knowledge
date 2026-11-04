# Verification recipes

_How to stand up and capture evidence on each surface — one page per stack, read before capturing and corrected after._

An agent that builds a mockup or changes something observable proves it on the real surface: a
screenshot, a video, the verbatim output of a request, a serial log. The procedure is the
[`cs-capture-evidence` skill](https://github.com/niway-dev/skills/tree/main/skills/cs-capture-evidence)
(`produce` entry); the decision is
[skills ADR 0017](https://github.com/niway-dev/skills/blob/main/docs/adr/0017-surface-evidence-generic-procedure-stack-recipes-in-the-hub.md).
These pages are the memory the procedure reads and writes.

## Three layers

| Layer | Holds | Lives in |
|---|---|---|
| Procedure | detect, stand up, capture, record, check secrets, deliver | the skill |
| Recipe | driver, stand-up and capture commands, traps, status | this folder |
| Scenario | login, seed data, ports, named states of one product | the project's `verify` skill, which links its recipe |

## Detect the stack

| Signal in the repo | Recipe |
|---|---|
| `package.json` with a web framework (Vite, Next, TanStack Start, Astro) or a lone `.html` mockup | [web-playwright](./web-playwright.md) |
| an HTTP server or a CLI whose output is the behaviour | [http-and-cli](./http-and-cli.md) |
| `package.json` with `electron` | [electron](./electron.md) |
| `app.json` / `expo`, `ios/`, `android/` | [mobile](./mobile.md) |
| `Package.swift` / `.xcodeproj`, `src-tauri/`, other native desktop | [native-desktop](./native-desktop.md) |
| `platformio.ini`, `sdkconfig`, Arduino sketches, M5Stack libraries | [embedded](./embedded.md) |

A monorepo with several surfaces uses the recipe of the surface the change touches.

## Status legend and write-back

Every recipe carries a status: **probed** (ran end to end, with the date and where), **partial**
(some steps ran), **unprobed** (believed, never run), **failed** (tried, did not work, and why).
A run that finds a recipe missing, wrong or `unprobed` tries the cheapest driver and writes the
result back here, whatever the outcome. A `failed` entry is as valuable as a `probed` one: it stops
the next session from trying the same thing.

## Rules for every surface

- Captures go to a git-ignored `.evidence/<branch-or-topic>/` and are never committed.
- Delivery: inline in T3 Code chat as `![alt](<absolute path>)`; PR with `gh pr edit <n> --attach
  <file>#<alt>` (gh ≥ 2.99, 10 MB per image); tracker issue through its attachment upload.
- Each capture states what is on screen, how it was reached, the commit, and what it does not prove.
- Check every image and output for secrets before delivery; redact or drop the evidence, never the claim.
- Text evidence stays text. An image of text only when asked.
- Layout: one row per state, the table grows down. `State | Before | After` for a visible change
  whose base can be stood up, `State | Capture` otherwise; `—` where a state has no "before". Never
  several states side by side: each extra column shrinks every image on a PR page.
- "Before" is the surface without the change: the commit before the first edit (for a stacked
  change, the commit the first layer starts from, since the layer below already carries part of it), or a
  capture taken before editing. A small change compared against a partly changed base looks
  identical, and the comparison proves nothing.
- An app is captured running for real. A render of its UI in a browser with fake data checks
  layout only; label it when it is the only evidence.
- An app with a window gets one capture at the size it opens with, no forced resize: a layout that
  breaks only at the launch size is invisible in captures taken at a chosen size.
- Stop every process the capture started.
