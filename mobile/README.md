# Mobile Knowledge Hub

Patterns and guidance for the Expo / React Native mobile app: app structure, how it
consumes the shared domain, auth, **running it** (dev builds, Metro, environment), and
**maps**.

## Contents

| File                             | Summary                                                                 |
| -------------------------------- | ----------------------------------------------------------------------- |
| [mobile-app.md](./mobile-app.md) | Expo/React Native app structure, tab navigation, shared domain package imports, and Better Auth integration. |
| [expo-dev-builds-and-metro.md](./expo-dev-builds-and-metro.md) | The two layers (native shell vs JS), Expo Go vs dev build, when to rebuild vs `bun dev`, Metro connection on emulators (`10.0.2.2` vs `--host localhost` + `adb reverse`), clean prebuild, environment setup, monorepo Metro config. |
| [google-maps.md](./google-maps.md) | `react-native-maps`: iOS Apple Maps vs Android Google Maps, why maps need a **dev build**, injecting the key via `app.config.ts`, the Google Cloud checklist (Maps SDK for Android + billing + restriction), the correct SHA-1 from Expo's keystore, reading logcat. |

> For **Convex as the backend** (client connection + Better Auth hosted in Convex), see
> the [`convex/`](../convex/) topic.
