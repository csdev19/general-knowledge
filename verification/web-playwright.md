# Web (TypeScript) and HTML mockups — Playwright

**Status:** probed (owner's Mac, 2026-11-02: `npx playwright@1.64.1 screenshot` of a `file://` mockup at both viewports; skills#45). Also probed 2026-11-02 on the same Mac: rest and hover shots of a mockup and of a real component in a Vite harness, with the project's `playwright-core` and the system Chrome, no browser download (`cs-capture-evidence/scripts/shots.mjs`).

## Stand up

- Mockup: open the file as `file://<absolute path>`, or `npx serve <dir>` when it loads assets.
- App: the repo's dev server on a free port (`--port`), in the background; record the PID. For CSS
  or package changes use the production preview ([local-preview](../web/local-preview.md)).
- Wait for the URL to answer before capturing.

## Capture

```bash
npx playwright screenshot --viewport-size=1440,900 --full-page <url> .evidence/<topic>/desktop.png
npx playwright screenshot --viewport-size=390,844 <url> .evidence/<topic>/mobile.png
```

Interaction (login, a click, a filled form) needs a short script with `page.screenshot`; video
with the `recordVideo` context option, only for flows of several steps.

### Without downloading a browser

A project that already depends on Playwright has `playwright-core`, and the machine usually has
Google Chrome. Launch that instead of a Playwright browser build:

```js
import { chromium } from "<path to the project's playwright-core>/index.mjs";
const browser = await chromium.launch({
  executablePath: "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome", // macOS
  headless: true,
});
```

The CLI form is `npx playwright screenshot --channel chrome …`. Where `playwright-core` lives
depends on the package manager: hoisted at `node_modules/playwright-core` (npm, yarn), under
`node_modules/.bun/playwright-core@<v>/node_modules/` (bun) or `node_modules/.pnpm/…` (pnpm). A
fresh linked worktree often has no `node_modules`: look in the main checkout. The
`cs-capture-evidence` skill ships `scripts/shots.mjs`, which does this lookup.

### States a static screenshot misses

- **Hover.** Shoot the element at rest with the pointer parked in a viewport corner outside it
  (read its box after scrolling it into view), then `locator.hover()` and shoot again as a page
  `clip` of its current box: a second locator screenshot can scroll and move it away from the
  pointer.
- **Animations.** Pausing CSS animations stops them wherever load left them, and rewinding to frame
  0 leaves entrance animations (fade-in, pop from `opacity: 0`) invisible. Right before each shot,
  `finish()` every finite animation and pause infinite ones at their start
  (`document.getAnimations()`); a scroll-driven timeline takes `CSS.percent(0)`, not milliseconds.
  Disable transitions with CSS.
- **A half-way state** (a progress bar at 40 %): skip the settling and pin it with CSS
  (`animation-play-state: paused; animation-delay: -4s`). That is exact only for an animation that
  starts after the CSS is injected.
- **A component inside a desktop or native shell** (an Electron renderer window): render the real
  component in a throwaway Vite page with the shell's bridge mocked, at the window's real size, and
  shoot that. It proves what the component renders, not the shell's behaviour.
- **Global type rules.** A base stylesheet that sets `font-weight` or `font-family` on every element
  can override what a `<button>` passes to its `<span>` label; check the label's computed weight
  when a shot looks off.

## Traps

- Browser revision: each Playwright version needs its own browser build. `npx playwright@latest`
  fails with "Executable doesn't exist … chromium_headless_shell-<rev>" when only an older build is
  cached. Use the system Chrome (above) before downloading anything; a download is the last resort
  (`npx playwright install chromium`, about 100 MB). Corrected 2026-11-02: this page used to
  recommend the download first; the system-Chrome launch worked without it.

- Hydration: capture after the page is idle (`waitForLoadState('networkidle')` or a test id), or
  the image shows a skeleton.
- A shared dev server the owner is running: pick another port, never kill theirs.
- Auth: reuse the project's E2E storage state ([e2e](../monorepos/testing/e2e.md)); that setup is
  what a project `verify` scenario is for.
