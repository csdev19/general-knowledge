# Web (TypeScript) and HTML mockups — Playwright

**Status:** probed (Chromium cached under `~/Library/Caches/ms-playwright` on the owner's Mac, 2026-11-02).

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

## Traps

- Hydration: capture after the page is idle (`waitForLoadState('networkidle')` or a test id), or
  the image shows a skeleton.
- A shared dev server the owner is running: pick another port, never kill theirs.
- Auth: reuse the project's E2E storage state ([e2e](../monorepos/testing/e2e.md)); that setup is
  what a project `verify` scenario is for.
