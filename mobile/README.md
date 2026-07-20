# Mobile Knowledge Hub

Patterns and guidance for the Expo / React Native mobile app: app structure, how it
consumes the shared domain, auth, **running it** (dev builds, Metro, environment),
**building and shipping it** (which command, which artifact, which environment), **maps**,
and **Android platform gotchas that only reproduce on a real device**.

## Contents

| File                             | Summary                                                                 |
| -------------------------------- | ----------------------------------------------------------------------- |
| [mobile-app.md](./mobile-app.md) | Expo/React Native app structure, tab navigation, shared domain package imports, and Better Auth integration. |
| [expo-dev-builds-and-metro.md](./expo-dev-builds-and-metro.md) | The two layers (native shell vs JS), Expo Go vs dev build, when to rebuild vs `bun dev`, Metro connection on emulators (`10.0.2.2` vs `--host localhost` + `adb reverse`), clean prebuild, environment setup, monorepo Metro config. |
| [expo-build-and-install-model.md](./expo-build-and-install-model.md) | Which command for which job: the two axes (debug/release × build profile), `run:*` vs `eas build`, local vs cloud EAS, one install per profile (id/name/scheme + the backend `trustedOrigins` coupling), where env vars actually travel (`.env` vs `eas.json` vs EAS-hosted) and the wrapper scripts that close both gaps, Gradle-heap OOM on release. |
| [expo-build-scripts.md](./expo-build-scripts.md) | **Copy-paste kit**: the complete working files — `app.config.ts` with `VARIANTS`, `eas.json`, both wrapper scripts (`run-android.sh`, `eas-local-build.sh`), the Gradle-heap config plugin, `package.json` scripts, `.gitignore` — each with what to rename and why it is written that way. |
| [google-maps.md](./google-maps.md) | `react-native-maps`: iOS Apple Maps vs Android Google Maps, why maps need a **dev build**, injecting the key via `app.config.ts`, the Google Cloud checklist (Maps SDK for Android + billing + restriction), the correct SHA-1 from Expo's keystore, reading logcat. |
| [android-edge-to-edge-keyboard.md](./android-edge-to-edge-keyboard.md) | **Device-only grey screen**: why `KeyboardAvoidingView` with `behavior="height"` collapses on Android 15+/16 (the IME stopped resizing the window), why an emulator on API ≤34 cannot reproduce it, the one-line fix, a 30-second detection snippet, and the debugging lessons. |

> For **Convex as the backend** (client connection + Better Auth hosted in Convex), see
> the [`convex/`](../convex/) topic.
