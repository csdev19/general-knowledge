# Expo build & install model

_Which command produces which artifact, for whom, wired to which backend. Covers the two
independent axes (debug/release × build profile), per-profile app identity so every
environment can be installed side by side, and where environment variables actually
travel. Stack-agnostic; assumes EAS for cloud builds._

> Prerequisite mental model: the **native shell vs JS bundle** split and when a rebuild is
> needed — see [expo-dev-builds-and-metro](./expo-dev-builds-and-metro.md). This document
> starts where that one ends: you know you need a build, now pick which.

## The two axes people conflate

Almost every "which command do I run" confusion comes from treating one axis as two.

| Axis | Values | Decides |
| --- | --- | --- |
| **Gradle/Xcode variant** | `debug` / `release` | Whether the JS bundle is **inside** the binary, whether minification and Hermes bytecode are on |
| **Build profile** | `development` / `preview` / `production` | **Identity** (name, application id, scheme) and **which backend** it talks to |

They compose freely: a `release` binary with `development` identity is perfectly normal
(a dev-backend app you can use unplugged). Name your scripts after the *combination*, not
after one axis, or you will keep re-deriving it.

**The `debug` consequence that surprises everyone:** a debug binary contains **no JS
bundle**. It fetches one from Metro at launch, so the app stops working the moment the
bundler stops or the cable comes out. That is the point during development (fast refresh),
and the reason it is the wrong artifact for "I want to use the app on my phone".

## The decision matrix

Two families. The question is not "APK or device?" — everything produces a binary — it is
**who is this for**.

**`run:*` — "put it on the device in front of me."** Compiles and installs over `adb`/
Xcode in one step, signed with the local debug keystore. Not distributable.

| Intent | Command shape |
| --- | --- |
| Daily iteration, fast refresh | `expo run:android` (debug, dev profile) |
| Reproduce a release-only bug (Hermes, minify, startup timing) | `expo run:android --variant release` |
| Use the real app on your own phone, unplugged, against prod | `--variant release` + the preview profile's env |

**`eas build` — "give me an artifact."** Signed with the **managed** keystore EAS holds,
so it is the same signature a store build carries.

| Intent | Command shape |
| --- | --- |
| A dev client to install once and reuse across JS changes | `eas build --profile development` |
| The QA build, without the cloud queue | `eas build --profile preview --local` |
| Validate the actual release artifact | `eas build --profile production --local` |
| Hand a link/QR to someone else | `eas build --profile preview` (cloud) |

**Rule of thumb: if it is for you, `run:*`; if it is for someone else, `eas build`.** The
exception is the **dev client** — worth building once even for yourself, because it is a
shell that loads whatever Metro serves, so it survives JS changes without recompiling.

### Local vs cloud EAS builds

The Gradle build underneath is identical. What differs is around it:

| | `--local` | Cloud |
| --- | --- | --- |
| Toolchain | yours (drifts when you upgrade a JDK) | pinned image, reproducible with CI |
| Env vars | your shell + whatever you plumb in | `eas.json` `env` + EAS-hosted variables **only** |
| Signing | pulls the managed credentials from EAS | same |
| Speed | slow first run, then Gradle cache; no queue | fresh every time, plus queue time |
| Artifact | a file on your disk | URL + QR on expo.dev |

Use local to iterate, cloud to distribute and for release automation. Signing is the one
difference with irreversible consequences: an AAB signed with the wrong keystore is
rejected by Play Store, and the keystore cannot be rotated without a support process.
`--local` avoids that trap by downloading the real credentials; a raw
`expo run:android --variant release` does **not** — it debug-signs.

## One install per profile

Without this, installing preview replaces your dev build and you cannot tell which is
which. Three things must differ:

| Profile | Launcher name | Application id | Scheme |
| --- | --- | --- | --- |
| `development` | `app-dev` | `com.you.app.dev` | `app-dev://` |
| `preview` | `app-preview` | `com.you.app.preview` | `app-preview://` |
| `production` | `App` | `com.you.app` | `app://` |

- **Application id** is what actually makes them coexist. Android (and iOS, via bundle id)
  keys installs by id — differing names alone still means each install replaces the last.
- **Name** is only so you can tell them apart in the launcher.
- **Scheme** keeps deep links and stored sessions separate. With a shared scheme the OS
  asks which app should handle a link, and an auth redirect can land in the wrong variant.

**Production keeps the bare id and scheme.** The id is immutable once published to a store,
and rotating the production scheme invalidates existing sessions.

```ts
// app.config.ts
const VARIANTS = {
  development: { idSuffix: ".dev", name: "app-dev", scheme: "app-dev" },
  preview: { idSuffix: ".preview", name: "app-preview", scheme: "app-preview" },
  production: { idSuffix: "", name: "App", scheme: "app" },
} as const;

// APP_VARIANT is set by your own scripts; EAS Build sets EAS_BUILD_PROFILE itself.
const requested = process.env.APP_VARIANT ?? process.env.EAS_BUILD_PROFILE ?? "development";
const variant = requested in VARIANTS ? requested : "development";
const { idSuffix, name, scheme } = VARIANTS[variant];

export default ({ config }) => ({
  ...config,
  name,
  scheme,
  ios: { ...config.ios, bundleIdentifier: `${BASE_ID}${idSuffix}` },
  android: { ...config.android, package: `${BASE_ID}${idSuffix}` },
});
```

Declare `APP_VARIANT` in **every** profile's `env` block in `eas.json` too, so a store
build never depends on `EAS_BUILD_PROFILE` being exported. If that assumption ever broke
you would ship a store app named `app-dev` with the wrong id.

Whatever keys the variant off, **`app.json` must not also declare `name`, `scheme` or
`version`** — one owner per value, or the two files drift and the winner is whichever the
merge order picks.

### Two couplings to remember

- **Switching profiles requires a clean prebuild.** The id and scheme are baked into
  `android/` at prebuild time and an incremental prebuild will not rewrite them —
  you silently reinstall the previous variant. Automate the detection (below).
- **Each scheme must be trusted by the backend** if auth derives its request Origin from
  it (`@better-auth/expo` does, via `Constants.expoConfig?.scheme`, which also seeds its
  `storagePrefix`). Adding a variant is a two-repo change, and **the backend must ship
  first** — an app whose scheme is not yet allowlisted cannot log in at all. The failure
  is invisible in Expo Go, which rides on `exp://`.

## Where environment variables actually travel

The single richest source of "works locally, broken in the build" bugs. Three sources,
different reach:

| Source | Reaches `expo run:*` | Reaches `eas build --local` | Reaches cloud EAS |
| --- | --- | --- | --- |
| `.env` in the app dir | ✅ Expo CLI loads it | ❌ **gitignored → not packaged** | ❌ |
| `env` block in `eas.json` | ❌ Expo CLI ignores `eas.json` | ✅ | ✅ |
| EAS-hosted variables (per environment) | ❌ | ✅ | ✅ |
| Your shell (`FOO=bar` prefix) | ✅ | ✅ (same process tree) | ❌ **never forwarded** |

Two traps follow directly:

1. **`.env` never reaches a cloud build.** EAS packages the project respecting
   `.gitignore`. A value that lives only in `.env` resolves to `undefined` on the worker.
   It is worst for values baked into the native manifest at prebuild (a maps API key):
   the app installs, runs, and hard-crashes the moment the relevant screen mounts — a
   native abort, so there is no JS error to find.
2. **`expo run:*` never reads `eas.json`.** Run the preview variant through it raw and you
   install an app *named* preview, with the preview id, talking to the **dev** backend.
   A mismatch that looks right and reports wrong.

Both are fixed by thin wrapper scripts rather than by remembering. The load-bearing
detail that makes them work: **`@expo/env` does not overwrite already-defined variables**
(`"is already defined and IS NOT overwritten"`), so anything exported before invoking
Expo wins over `.env`, and `.env` still fills the gaps.

### Wrapper 1 — run a profile on a device, with its real backend

```sh
#!/bin/sh
# usage: run-android.sh <profile> <debug|release>
set -e
profile="$1"; gradle_variant="$2"
export APP_VARIANT="$profile"

# The profile's backend, straight from eas.json — never duplicated, so a local install
# and a cloud build of the same profile cannot drift.
eval "$(node -e "
  const env = (require('./eas.json').build['$profile'] || {}).env || {};
  for (const [k, v] of Object.entries(env)) console.log(\`export \${k}='\${v}'\`);
")"

# The id is baked into android/; an incremental prebuild will not rewrite it.
expected="$(bunx expo config --type public 2>/dev/null | sed -n "s/.*package: [^']*'\([^']*\)'.*/\1/p" | head -1)"
current="$(sed -n "s/.*applicationId ['\"]\([^'\"]*\)['\"].*/\1/p" android/app/build.gradle 2>/dev/null | head -1)"
[ -n "$current" ] && [ "$current" != "$expected" ] && bunx expo prebuild --clean --platform android

exec bunx expo run:android --variant "$gradle_variant"
```

### Wrapper 2 — a local EAS build that can see `.env`

```sh
#!/bin/sh
# usage: eas-local-build.sh <platform> <profile>
set -e
platform="$1"; profile="$2"

# Caller's exports win over the file: snapshot, source, re-apply.
caller_env="$(export -p)"
set -a; [ -f .env ] && . ./.env; set +a
eval "$caller_env"

# Drop exactly what this profile declares in eas.json — read, not hardcoded, because a
# `development` profile deliberately declares none and is meant to run on .env.
owned="$(node -p "Object.keys((require('./eas.json').build['$profile']||{}).env||{}).join(' ')")"
[ -n "$owned" ] && unset $owned

export APP_VARIANT="$profile"
exec bunx eas-cli build --platform "$platform" --profile "$profile" --local
```

Note the asymmetry: wrapper 1 *adds* `eas.json` values because Expo CLI cannot see them;
wrapper 2 *removes* them from `.env` because EAS applies them itself and the file must not
shadow them. Same goal — the profile's declaration wins — from opposite directions.

## Gotchas worth pre-empting

**Gradle heap on release builds.** `:app:mergeDexRelease` dies with
`java.lang.OutOfMemoryError: Java heap space` once the app has enough native dependencies:
D8 runs inside the Gradle daemon JVM and the Expo template ships `-Xmx2048m`. Debug builds
merge far fewer dex archives, so only release surfaces it. Fix it in a **config plugin**,
not in `android/gradle.properties` — that directory is gitignored and regenerated by every
prebuild, and `eas build --local` prebuilds into a temp directory, so a hand edit is
discarded before Gradle reads it:

```js
const { withGradleProperties } = require("expo/config-plugins");
module.exports = (config) =>
  withGradleProperties(config, (c) => {
    const value = process.env.GRADLE_JVM_ARGS || "-Xmx4096m -XX:MaxMetaspaceSize=1024m";
    const entry = { type: "property", key: "org.gradle.jvmargs", value };
    const i = c.modResults.findIndex((p) => p.key === "org.gradle.jvmargs");
    i === -1 ? c.modResults.push(entry) : (c.modResults[i] = entry);
    return c;
  });
```

Keep the default modest (4 GB, not 8): the same plugin runs on EAS workers, where
over-requesting risks the daemon failing to start. Override per build with the env var.

**Verify a manifest value instead of guessing.** Anything injected at prebuild can be read
back out of the artifact:

```sh
aapt2 dump xmltree app.apk --file AndroidManifest.xml | grep -A1 geo.API_KEY
```

**Signing determines more than distribution.** API key restrictions (Google Maps and
friends) are keyed on *package + signing SHA-1*. With per-profile ids and two keystores
(debug for `run:*`, EAS-managed for builds), each **pair** you actually build has to be
allowlisted. A grey map instead of a crash means the key arrived and the restriction
rejected it.

**Ignore build artifacts.** `eas build --local` drops a ~100 MB APK/AAB in the project
directory. Add `*.apk` / `*.aab` to `.gitignore` on day one.

## Checklist for a new project

1. `VARIANTS` in `app.config.ts`; remove `name` / `scheme` / `version` from `app.json`.
2. `APP_VARIANT` declared in every `eas.json` profile's `env`.
3. Backend allowlists every variant scheme, shipped **before** the app.
4. Wrapper scripts for both directions of the env plumbing; no raw `expo run:*` script
   left in `package.json` to bypass them.
5. Gradle-heap config plugin.
6. `*.apk` / `*.aab` gitignored.
7. Each package + keystore SHA-1 pair registered with whichever API restricts by them.

## Related

- [expo-dev-builds-and-metro](./expo-dev-builds-and-metro.md) — the native/JS layer split,
  when a rebuild is required, Metro connectivity, clean prebuild, toolchain setup.
- [google-maps](./google-maps.md) — the maps key, the Google Cloud checklist and SHA-1.
- [monorepos/release-please](../monorepos/) — per-component versioning, which decides when
  a store build is even cut.
