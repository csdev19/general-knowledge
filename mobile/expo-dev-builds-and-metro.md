# Expo dev builds & Metro

_The operational model for running an Expo app: the two layers, Expo Go vs dev build,
when to rebuild vs reload, Metro connection on emulators, and environment setup. Stack-
agnostic — applies to any Expo SDK 57 app._

## The two layers (the mental model that removes 90% of the confusion)

A React Native app is **two separate things**:

```
┌─ NATIVE SHELL (the APK/IPA) ──────────────────┐  ← compiled Java/Kotlin/C++/Swift
│  native modules, AndroidManifest, permissions, │    icons, splash, the maps API key
│  ┌─ JS BUNDLE ─────────────────────────────┐   │  ← your React/TS, runs inside the shell
│  │  components, styling, logic              │   │
│  └─────────────────────────────────────────┘   │
└────────────────────────────────────────────────┘
```

| | `expo run:android` / `run:ios` | `expo start` (`bun dev`) |
| --- | --- | --- |
| Builds | the **native shell** (prebuild → Gradle/Xcode → binary), installs it | starts **Metro**, serves the **JS bundle** |
| Includes | native modules, manifest, maps key, permissions, icons | just JS/TS |
| Speed | slow (minutes; compiles native code) | fast (seconds) |
| When | the **native** layer changed | the **JS** layer changed (daily) |

They are **layers, not modes** — you need both: `run` once to install the shell, then
`start` to feed JS into it.

## When do you need to rebuild?

| Change | Rebuild? |
| --- | --- |
| JS/TS, components, styling | ❌ No — `bun dev` + fast refresh |
| A dependency with **native code** (expo-network, react-native-maps, reanimated…) | ✅ Yes |
| Native config (`app.json`/`app.config`, maps key, permissions, scheme, icons, splash, config plugins) | ✅ Yes (baked into the native manifest at prebuild) |
| SDK bump | ✅ Yes |

**Golden rule:** touched something outside your JS (`app.json`, `.env` native vars, a
native dep) → **rebuild**. Touched only JS/TS → **`bun dev`**.

> Local dev vs prod is the **same** native shell in two variants: local = a *debug*
> build whose JS is served live by Metro (fast refresh, dev menu); prod = a *release*
> build with the JS **bundled into the binary** (no Metro). The shell compiles the same
> way; only debug→release and where the JS lives differ.

## Expo Go vs dev build — pick the right one

- **Expo Go**: a generic host app. Only works with the modules Expo bundles, and **cannot
  inject your native config** (its own scheme, its own manifest). On an **emulator/
  simulator, `expo start` auto-installs the SDK-matching Expo Go** — so a plain SDK 57 app
  runs there even when the store's Expo Go lags. On a **physical device** you're stuck
  with the store version until it supports your SDK.
- **Dev build** (`expo run:*` or EAS `development` profile): **your own** Expo Go, compiled
  with your native modules + config. Required the moment you use a native module Expo Go
  doesn't bundle, or need your own native config (**Google Maps key**, custom plugins).
- **`react-native-maps` on Android needs a dev build** — Expo Go uses its own (currently
  failing) Google key and ignores yours. See [google-maps](./google-maps.md).

**Don't keep Expo Go installed next to a dev build** on the same emulator: it hijacks
`a`/deep-links and you end up in the wrong app. `adb uninstall host.exp.exponent` (it
reinstalls trivially).

## Metro connection on an Android emulator

The finicky part. Two working setups — don't mix:

| App | Metro host | How the app reaches Metro |
| --- | --- | --- |
| **Dev build** | **default** (`expo start`, binds `0.0.0.0`) | app connects via `10.0.2.2:8081` (emulator alias for host loopback) — works because Metro is on all interfaces |
| **Expo Go** | `expo start --host localhost` + `adb reverse tcp:8081 tcp:8081` | app connects via `localhost:8081`, tunneled by `adb reverse` |

- **Dev build: do NOT use `--host localhost`** — it binds Metro to `127.0.0.1` only, and
  the dev build's `10.0.2.2` connection gets `ECONNREFUSED`. Plain `bun dev` is correct.
- The LAN-IP default (`192.168.x.x`) is flaky on emulators; the two rows above are the
  reliable setups.
- **"Unable to load script"** = Metro isn't up / not reachable. Ensure Metro shows
  `Waiting on http://localhost:8081`, `adb reverse` is set, then RELOAD on device (or `r`).
- **"Cannot connect to Expo CLI"** = the app points at an old/unreachable dev-server URL
  (often a stale LAN IP). Force-stop the app and reopen from the current `expo start`.

## When native gets stale — clean prebuild

`expo run:android` runs an **incremental** prebuild that can skip re-applying config-plugin
mods (e.g. the Google Maps `<meta-data>` didn't land). A **clean** prebuild regenerates
`android/`/`ios/` and re-runs every mod:

```bash
npx expo prebuild --clean   # regenerates native dirs from app.config + plugins
npx expo run:android        # then build/install
```

## Environment setup (one-time, macOS)

Android tooling isn't on `PATH` by default — add to `~/.zshrc` (changes only apply to
**new** terminals):

```bash
export ANDROID_HOME="$HOME/Library/Android/sdk"
export JAVA_HOME="/Applications/Android Studio.app/Contents/jbr/Contents/Home"  # bundled JDK
export PATH="$JAVA_HOME/bin:$ANDROID_HOME/platform-tools:$ANDROID_HOME/emulator:$PATH"
```

Gradle needs Java — "Unable to locate a Java Runtime" means the shell predates this edit
(`source ~/.zshrc` or open a new terminal). Emulator: `emulator -avd <name> -no-snapshot`
(a corrupt `default_boot` snapshot hangs the emulator `Offline`; `-no-snapshot` cold-boots).

- **iOS simulator**: what matters is the **iOS runtime version**, not the phone model.
  SDK 57 needs iOS 16+; use the newest runtime your Xcode ships (Xcode 16.x → iOS 18.6).
  iOS 26 features (e.g. `expo-glass-effect` liquid glass) need Xcode 26.

<a id="monorepo-metro-config"></a>
## Monorepo Metro config

For Metro to resolve workspace packages (`@scope/backend`, `@scope/domain`) from source
and the hoisted root `node_modules`:

```js
// apps/mobile/metro.config.js
const { getDefaultConfig } = require("expo/metro-config");
const path = require("path");
const projectRoot = __dirname;
const workspaceRoot = path.resolve(projectRoot, "../..");
const config = getDefaultConfig(projectRoot);
config.watchFolders = [workspaceRoot];
config.resolver.nodeModulesPaths = [
  path.resolve(projectRoot, "node_modules"),
  path.resolve(workspaceRoot, "node_modules"),
];
module.exports = config;
```

## Related

- [Google Maps on mobile](./google-maps.md) — the maps-specific build + key setup.
- [Convex client connection](../convex/client-connection.md) — what the JS connects to.
