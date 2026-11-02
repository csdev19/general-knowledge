# Electron — Playwright `_electron`

**Status:** probed (kaipu-record#398, scenarios ran).

```ts
import { _electron as electron } from 'playwright'
const app = await electron.launch({ args: ['.'], recordVideo: { dir: '.evidence/<topic>' } })
const win = await app.firstWindow()
await win.screenshot({ path: '.evidence/<topic>/main.png' })
await app.close()
```

Runs unattended and needs no Screen Recording permission. Build the main process first if the repo
compiles it. OS dialogs and permission prompts are outside the window: they cannot be captured this
way and the proof row says so.
