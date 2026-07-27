# Expo build scripts (copy-paste kit)

_The complete, working files behind
[expo-build-and-install-model](./expo-build-and-install-model.md): per-profile identity,
env plumbing in both directions, and the Gradle-heap plugin. Copied **verbatim** from a
shipping Expo SDK 57 + EAS monorepo (`trip-planner`), not retyped — each one is in daily
use. Every section ends with what to rename._

Layout the files assume:

```
apps/native/
├── app.json                       # static config; NO name / scheme / version keys
├── app.config.ts                  # identity per profile + plugin registration
├── eas.json                       # per-profile env, incl. APP_VARIANT
├── plugins/
│   └── with-gradle-memory.js
└── scripts/
    ├── run-android.sh             # install a profile on a device
    └── eas-local-build.sh         # produce a signed artifact locally
```

---

## 1. `app.config.ts` — identity per profile

Owns the name, application id, scheme, version and plugin list. What makes three installs
coexist on one device.

```ts
import type { ConfigContext, ExpoConfig } from "expo/config";
import pkg from "./package.json";

/**
 * Dynamic config layered on top of app.json. Four jobs:
 *
 * 1. `version` comes from package.json — the single source of truth that
 *    Release Please bumps on a native release. app.json deliberately has NO
 *    `version` key so the two can never drift. This is the store-visible
 *    marketing version; `versionCode`/`buildNumber` are a different number,
 *    owned by EAS (`appVersionSource: "remote"` + `autoIncrement`).
 *
 * 2. Inject the Google Maps Android API key (read from
 *    EXPO_PUBLIC_GOOGLE_MAPS_API_KEY in apps/native/.env) into
 *    `android.config.googleMaps.apiKey`, which react-native-maps turns into
 *    the `com.google.android.geo.API_KEY` <meta-data> in AndroidManifest.
 *    Keeping it in .env avoids committing the key. Requires a rebuild
 *    (`expo run:android`) because the key is baked into the native manifest
 *    at prebuild time.
 *
 * 3. Register `plugins/with-gradle-memory`, which raises the Gradle daemon heap
 *    over the Expo template default — 2 GB is not enough for
 *    `:app:mergeDexRelease` and release builds die with an OOM in D8.
 *
 * 4. Give each build profile its own app name, application id and URL scheme, so
 *    dev, preview and production installs coexist on one device instead of
 *    overwriting each other, and their deep links and sessions stay separate.
 *    See VARIANTS below.
 */

/**
 * Per-profile identity.
 *
 * `idSuffix` MUST differ for the installs to coexist — Android keys installs by
 * application id, so a shared id means the newest install replaces the previous
 * one no matter what it is called. The name only makes them distinguishable in
 * the launcher.
 *
 * `scheme` must differ too, for a subtler reason: with a shared scheme, Android
 * asks the user which app should handle a `ripuy://` deep link, and an auth
 * redirect can land in the wrong variant. It is also what @better-auth/expo
 * derives its request Origin and its `storagePrefix` from
 * (`src/lib/auth-client.ts`), so a per-variant scheme additionally keeps each
 * variant's stored session separate.
 *
 * EVERY scheme here MUST be listed in `trustedOrigins` in
 * `packages/backend/convex/auth.ts` — Better-Auth rejects logins from an
 * unlisted origin, and standalone builds are the only place this shows up
 * (Expo Go rides on `exp://`).
 *
 * Production keeps the bare id and scheme: the id is immutable once published to
 * a store, and changing the production scheme would invalidate existing
 * sessions. Only the non-production variants are suffixed.
 */
const VARIANTS = {
  development: { idSuffix: ".dev", name: "ripuy-dev", scheme: "ripuy-dev" },
  preview: { idSuffix: ".preview", name: "ripuy-preview", scheme: "ripuy-preview" },
  production: { idSuffix: "", name: "Ripuy", scheme: "ripuy" },
} as const;

type VariantName = keyof typeof VARIANTS;

/**
 * Which variant to build.
 *
 * - `APP_VARIANT` — set explicitly by `scripts/eas-local-build.sh`, and what you
 *   pass by hand for a one-off (`APP_VARIANT=preview bun run rebuild:android`).
 * - `EAS_BUILD_PROFILE` — set by EAS Build itself, so cloud builds are labelled
 *   correctly without any extra wiring.
 * - Neither set means a plain local `expo run:android`, which is development.
 *
 * An unknown value falls back to development rather than throwing: the failure
 * mode of a mislabelled dev build is a confusing launcher icon, while throwing
 * would break `expo config` for anyone with a stray env var.
 */
const requested = process.env.APP_VARIANT ?? process.env.EAS_BUILD_PROFILE ?? "development";
const variant: VariantName = requested in VARIANTS ? (requested as VariantName) : "development";
const { idSuffix, name, scheme } = VARIANTS[variant];

export default ({ config }: ConfigContext): ExpoConfig => ({
  ...config,
  name,
  scheme,
  slug: config.slug ?? "ripuy-trip-planner",
  version: pkg.version,
  // Appended here, not in app.json, so the static plugin list stays declarative
  // and this stays next to the other build-time concerns above.
  plugins: [...(config.plugins ?? []), "./plugins/with-gradle-memory"],
  ios: {
    ...config.ios,
    bundleIdentifier: `${config.ios?.bundleIdentifier ?? "com.csdev19.ripuy"}${idSuffix}`,
  },
  android: {
    ...config.android,
    package: `${config.android?.package ?? "com.csdev19.ripuy"}${idSuffix}`,
    config: {
      ...config.android?.config,
      googleMaps: {
        apiKey: process.env.EXPO_PUBLIC_GOOGLE_MAPS_API_KEY,
      },
    },
  },
});
```

**Adapt:** `VARIANTS` names/schemes, the `com.csdev19.ripuy` fallbacks, `slug`. Drop the
`googleMaps` block if the app has no maps.

**Non-obvious bits:**

- The fallback chain is ordered so the *safe* value wins when nothing is set: a bare
  `expo run:android` builds **development**. Defaulting to production instead would mean a
  local run overwriting a store install and claiming the production id.
- Unknown `APP_VARIANT` silently falls back rather than throwing — a stray env var should
  not break `expo config`, which tooling calls constantly.
- Verify all paths before trusting it:

  ```sh
  for v in "" development preview production garbage; do
    printf "%-12s " "${v:-<unset>}"
    APP_VARIANT="$v" bunx expo config --type public | grep -E "^  name:|^  scheme:|^    package:"
  done
  ```

---

## 2. `app.json` — what must NOT be there

```json
{
  "expo": {
    "slug": "ripuy-trip-planner",
    "owner": "csdev19",
    "ios": { "bundleIdentifier": "com.csdev19.ripuy" },
    "android": { "package": "com.csdev19.ripuy" }
  }
}
```

No `name`, no `scheme`, no `version` — `app.config.ts` computes all three. Leaving a dead
copy here is how the two files drift, and the loser is whichever the merge order picks.
The `bundleIdentifier` / `package` that stay are the **base** the suffix appends to.

---

## 3. `eas.json` — per-profile env

```json
{
  "cli": { "version": ">= 21.0.0", "appVersionSource": "remote" },
  "build": {
    "development": {
      "developmentClient": true,
      "distribution": "internal",
      "environment": "development",
      "android": { "buildType": "apk" },
      "env": { "APP_VARIANT": "development" }
    },
    "preview": {
      "distribution": "internal",
      "environment": "preview",
      "android": { "buildType": "apk" },
      "env": {
        "APP_VARIANT": "preview",
        "EXPO_PUBLIC_CONVEX_URL": "https://<prod>.convex.cloud",
        "EXPO_PUBLIC_CONVEX_SITE_URL": "https://<prod>.convex.site",
        "EXPO_PUBLIC_WEB_BASE_URL": "https://example.app"
      }
    },
    "production": {
      "autoIncrement": true,
      "environment": "production",
      "android": { "buildType": "app-bundle" },
      "env": {
        "APP_VARIANT": "production",
        "EXPO_PUBLIC_CONVEX_URL": "https://<prod>.convex.cloud",
        "EXPO_PUBLIC_CONVEX_SITE_URL": "https://<prod>.convex.site",
        "EXPO_PUBLIC_WEB_BASE_URL": "https://example.app"
      }
    }
  }
}
```

**Why `APP_VARIANT` is declared here even though `EAS_BUILD_PROFILE` exists:** the fallback
would work, but it makes a store build's identity depend on an EAS implementation detail.
If that ever changed you would ship a store app named `-dev` with the wrong id — silently,
and only discoverable after upload. One line per profile removes the dependency.

**`development` declares no backend URLs on purpose.** It is the profile meant to run
against your local `.env`. This is exactly why the scripts below read the owned-key list
from `eas.json` instead of hardcoding it — see script 5.

**Secrets do not go here.** Values that must not be committed (an API key you rotate per
environment) belong in EAS-hosted environment variables:

```sh
bunx eas-cli env:create --environment preview \
  --name EXPO_PUBLIC_GOOGLE_MAPS_API_KEY --value <key> \
  --visibility sensitive --type string --non-interactive --force
```

`--force` makes it an upsert, so it is safe to script — e.g. a CI step that pushes the
value from a GitHub secret, keeping GitHub as the source of truth and EAS as a mirror.

---

## 4. `scripts/run-android.sh` — install a profile on a device

Solves: *`expo run:*` never reads `eas.json`*, so the preview variant would install with
the dev backend.

```sh
#!/bin/sh
# Build and install on a connected device/emulator, with a profile's identity
# and backend.
#
# Usage: ./scripts/run-android.sh <profile> <debug|release>
#
# WHY THIS EXISTS
# `expo run:android` knows nothing about eas.json — it only loads .env, which
# points at the dev Convex deployment. Running the preview variant through it
# raw would install an app called "ripuy-preview", with the preview application
# id, talking to the *dev* backend: a mismatch that looks right and reports
# wrong. This exports the profile's `env` block from eas.json first, so the
# identity and the backend always agree.
#
# @expo/env does not overwrite variables that are already defined, so what is
# exported here wins over .env, and .env still supplies everything eas.json
# does not carry (notably EXPO_PUBLIC_GOOGLE_MAPS_API_KEY).
set -e

profile="$1"
gradle_variant="$2"

if [ -z "$profile" ] || [ -z "$gradle_variant" ]; then
  echo "usage: $0 <profile> <debug|release>" >&2
  exit 1
fi

APP_VARIANT="$profile"
export APP_VARIANT

# The profile's backend, straight from eas.json — never duplicated here, so a
# local install and a cloud build of the same profile cannot drift apart.
eval "$(node -e "
  const env = (require('./eas.json').build['$profile'] || {}).env || {};
  for (const [key, value] of Object.entries(env)) {
    console.log(\`export \${key}='\${value}'\`);
  }
")"

# The application id and scheme are baked into android/ at prebuild time, and a
# non-clean prebuild will not rewrite them — switching profiles without this
# would silently reinstall the previous variant. Detect the mismatch and clean.
# `[^']*` before the opening quote skips the ANSI colour codes Expo emits; the
# generated build.gradle quotes the id with ', but accept " too.
expected_id="$(bunx expo config --type public 2>/dev/null | sed -n "s/.*package: [^']*'\([^']*\)'.*/\1/p" | head -1)"
current_id="$(sed -n "s/.*applicationId ['\"]\([^'\"]*\)['\"].*/\1/p" android/app/build.gradle 2>/dev/null | head -1)"

if [ -n "$current_id" ] && [ -n "$expected_id" ] && [ "$current_id" != "$expected_id" ]; then
  echo "==> android/ holds $current_id but $profile needs $expected_id — running a clean prebuild"
  bunx expo prebuild --clean --platform android
fi

exec bunx expo run:android --variant "$gradle_variant"
```

**Adapt:** `bunx` → your package runner. Nothing else is project-specific.

**Why each part is the way it is:**

- **`eval` of generated `export` lines** rather than a `--env-file`: `expo run:*` has no
  such flag, and reading `eas.json` keeps a single declaration of each profile's backend.
  Duplicating the URLs in a script is how a local install and a cloud build of the "same"
  profile end up pointing at different backends.
- **The precedence is load-bearing.** `@expo/env` logs
  `"<key>" is already defined and IS NOT overwritten` — verify it in
  `node_modules/@expo/env/build/index.js` if you ever doubt it. Exporting first is what
  makes `eas.json` beat `.env`, and `.env` still fills what `eas.json` omits.
- **`[^']*` in the sed** skips Expo's ANSI colour codes; `NO_COLOR=1` does **not** strip
  them from `expo config`. The generated `build.gradle` quotes the id with `'`, but the
  pattern accepts `"` in case a template changes.
- **The auto-clean is not a nicety.** An incremental prebuild leaves the old
  `applicationId` in place, so without it, switching profiles reinstalls the *previous*
  variant while every log line claims otherwise. Cost: a full native rebuild on each
  switch.
- **`exec`** replaces the shell so Ctrl-C reaches Metro instead of orphaning it.

---

## 5. `scripts/eas-local-build.sh` — a local EAS build that can see `.env`

Solves: *`eas build --local` packages the project respecting `.gitignore`*, so `.env`
never reaches it.

```sh
#!/bin/sh
# Local EAS build with the values from apps/native/.env.
#
# Usage: ./scripts/eas-local-build.sh <platform> <profile>
#
# WHY THIS EXISTS
# `eas build --local` packages the project the same way a cloud build does —
# respecting .gitignore — and `.env` is gitignored, so it never reaches the
# build. Every EXPO_PUBLIC_* value would resolve to undefined, and the one that
# hurts most is EXPO_PUBLIC_GOOGLE_MAPS_API_KEY: it is baked into
# AndroidManifest.xml at prebuild time, so a build without it installs fine and
# then hard-crashes the moment a map screen mounts.
#
# `expo run:android` does NOT need this — Expo CLI loads .env itself.
#
# WHY THE UNSET
# eas.json's `env` block owns which backend each profile talks to, and .env
# points at the *dev* Convex deployment. Sourcing .env wholesale could shadow
# those and silently produce a "preview" build wired to dev — the exact
# mismatch the preview profile exists to catch. Whatever eas.json declares must
# stay authoritative, so those keys are dropped after sourcing. Everything else
# in .env flows through, no per-variable maintenance.
#
# The dropped keys are read from eas.json per profile rather than hardcoded:
# the `development` profile deliberately declares no `env` block — it is meant
# to run against the local .env — so a fixed list would blank out its Convex
# URLs and the app would throw on launch.
set -e

platform="$1"
profile="$2"

if [ -z "$platform" ] || [ -z "$profile" ]; then
  echo "usage: $0 <platform> <profile>" >&2
  exit 1
fi

# Snapshot what the caller exported, source .env, then re-apply the snapshot so
# an inline `FOO=bar bun run ...` override wins over the file instead of being
# silently clobbered by it.
caller_env="$(export -p)"
set -a
[ -f .env ] && . ./.env
set +a
eval "$caller_env"

# Owned by the `env` block of this profile in eas.json — see above.
owned_by_eas_json="$(node -p "Object.keys((require('./eas.json').build['$profile'] || {}).env || {}).join(' ')")"
if [ -n "$owned_by_eas_json" ]; then
  # shellcheck disable=SC2086 # word splitting is the point: unset takes a list
  unset $owned_by_eas_json
fi

if [ -z "$EXPO_PUBLIC_GOOGLE_MAPS_API_KEY" ]; then
  echo "warning: EXPO_PUBLIC_GOOGLE_MAPS_API_KEY is unset — map screens will crash this build." >&2
fi

# Names the app "ripuy-dev" / "ripuy-preview" and suffixes the application id so
# this install does not replace another profile's. See VARIANTS in app.config.ts.
APP_VARIANT="$profile"
export APP_VARIANT

exec bunx eas-cli build --platform "$platform" --profile "$profile" --local
```

**Adapt:** the `EXPO_PUBLIC_GOOGLE_MAPS_API_KEY` warning — point it at whichever value
your app cannot survive without. (Keep one. A named warning up front beats a 14-minute
build that installs and then crashes.)

**The two subtleties:**

- **`export -p` snapshot / re-apply.** `set -a; . ./.env` assigns unconditionally, so
  without the snapshot the file would clobber an inline
  `FOO=bar bun run build:local:...`, which is the opposite of what anyone expects. Verify:

  ```sh
  EXPO_PUBLIC_GOOGLE_MAPS_API_KEY=OVERRIDE ./scripts/eas-local-build.sh android preview
  # → must use OVERRIDE, not the .env value
  ```

- **The unset list is read, not written.** Hardcoding it broke `development`, which
  deliberately declares no `env` block: the fixed list blanked its Convex URLs and the app
  threw on launch. Reading per profile means adding a variable to `eas.json` needs no
  script change.

Note the symmetry with script 4: one **adds** `eas.json` values because Expo CLI cannot
see them, the other **removes** them from `.env` because EAS applies them itself. Same
rule — the profile's declaration wins — approached from opposite sides.

---

## 6. `plugins/with-gradle-memory.js` — the release-build OOM

```js
const { withGradleProperties } = require("expo/config-plugins");

/**
 * Raise the Gradle daemon heap above the Expo template default.
 *
 * The template ships `-Xmx2048m`, which is not enough for `:app:mergeDexRelease`
 * on this app: D8 runs inside the daemon JVM (no worker isolation) and dies with
 * `java.lang.OutOfMemoryError: Java heap space` roughly 14 minutes into a release
 * build. Debug builds merge far fewer dex archives and stay under the limit,
 * which is why only release surfaced it.
 *
 * This lives in a config plugin rather than in `android/gradle.properties`
 * because `android/` is gitignored and regenerated by every prebuild — and
 * `eas build --local` prebuilds into a throwaway temp directory, so a hand edit
 * there survives nothing. The plugin applies on every prebuild path: local,
 * `--local`, and EAS workers.
 *
 * Override for a one-off build with GRADLE_JVM_ARGS, e.g.
 *   GRADLE_JVM_ARGS="-Xmx8192m -XX:MaxMetaspaceSize=1024m" bun run rebuild:android:release
 */
const DEFAULT_JVM_ARGS = "-Xmx4096m -XX:MaxMetaspaceSize=1024m";

module.exports = function withGradleMemory(config) {
  return withGradleProperties(config, (gradleConfig) => {
    const value = process.env.GRADLE_JVM_ARGS || DEFAULT_JVM_ARGS;
    const entry = { type: "property", key: "org.gradle.jvmargs", value };
    const properties = gradleConfig.modResults;
    const index = properties.findIndex(
      (property) => property.type === "property" && property.key === "org.gradle.jvmargs",
    );

    if (index === -1) {
      properties.push(entry);
    } else {
      properties[index] = entry;
    }

    return gradleConfig;
  });
};
```

**Adapt:** nothing. Register it in `app.config.ts` (`plugins: [..., "./plugins/with-gradle-memory"]`)
and verify:

```sh
bunx expo prebuild --platform android
grep org.gradle.jvmargs android/gradle.properties   # → -Xmx4096m -XX:MaxMetaspaceSize=1024m
```

Keep the default at 4 GB even on a 32 GB machine: the same plugin runs on EAS workers,
where over-requesting risks the daemon failing to start. Raise per build with the env var.

`withGradleProperties` is the pattern for **any** `gradle.properties` value —
`android.enableJetifier`, `newArchEnabled`, `reactNativeArchitectures` — that has to
survive a regenerated `android/`.

---

## 7. `package.json` scripts

```json
{
  "scripts": {
    "rebuild:android": "./scripts/run-android.sh development debug",
    "rebuild:android:release": "./scripts/run-android.sh development release",
    "rebuild:android:preview": "./scripts/run-android.sh preview release",

    "build:local:android:development": "./scripts/eas-local-build.sh android development",
    "build:local:android:preview": "./scripts/eas-local-build.sh android preview",
    "build:local:android:production": "./scripts/eas-local-build.sh android production",

    "build:android:preview": "bunx eas-cli build --platform android --profile preview",
    "build:android:production": "bunx eas-cli build --platform android --profile production",

    "prebuild:clean": "expo prebuild --clean"
  }
}
```

Naming convention that pays for itself: **`rebuild:*` installs on your device,
`build:*` produces an artifact, `:local` means compiled here.**

**Delete any bare `"android": "expo run:android"`.** It bypasses the variant, the profile's
backend and the clean detection — and it is the one people reach for out of habit. An
escape hatch nobody remembers is a hatch that gets used by accident.

`chmod +x scripts/*.sh` and commit the bit, or every fresh clone fails on
`permission denied`.

---

## 8. `.gitignore`

```gitignore
/android
/ios
.env

# Local build artifacts (eas build --local writes these here)
*.apk
*.aab
```

`eas build --local` drops a ~100 MB APK/AAB in the project root. Add this before the first
local build, not after you have already staged one.

---

## Verification, end to end

Cheap checks that catch the failure modes above without waiting for a full build:

```sh
# Identity resolves for every profile, including the unset and garbage cases
for v in "" development preview production garbage; do
  APP_VARIANT="$v" bunx expo config --type public | grep -E "^  name:|^  scheme:|^    package:"
done

# Each profile's owned env keys are what you expect (development: empty but APP_VARIANT)
for p in development preview production; do
  node -p "Object.keys((require('./eas.json').build['$p']||{}).env||{}).join(' ')"
done

# The Gradle heap made it into the generated project
grep org.gradle.jvmargs android/gradle.properties

# A value injected at prebuild made it into the artifact
aapt2 dump xmltree <build>.apk --file AndroidManifest.xml | grep -A1 geo.API_KEY
```

## Related

- [expo-build-and-install-model](./expo-build-and-install-model.md) — the reasoning these
  files implement: the two axes, the decision matrix, where env vars travel.
- [expo-dev-builds-and-metro](./expo-dev-builds-and-metro.md) — native/JS layers, when a
  rebuild is required, Metro connectivity.
- [google-maps](./google-maps.md) — the key, the Google Cloud checklist, the SHA-1 pairs.
