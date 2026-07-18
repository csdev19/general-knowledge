# Maps on mobile (react-native-maps)

_Getting `react-native-maps` to actually render on Android — the platform differences,
the dev-build requirement, the API key wiring, and the Google Cloud setup that trips
everyone up._

## iOS "just works", Android doesn't — here's why

`react-native-maps` uses a **different map per platform** when no `provider` is set:

| Platform | Map used | API key needed? |
| --- | --- | --- |
| **iOS** (no `provider={PROVIDER_GOOGLE}`) | **Apple Maps** (built into iOS) | ❌ No — nothing to configure |
| **Android** | **Google Maps** (always; no Apple Maps on Android) | ✅ Yes, always |

So an unconfigured map renders fine on iOS (Apple Maps) and blank on Android (missing/
misconfigured Google key). Setting `provider={PROVIDER_GOOGLE}` opts iOS into Google too
(needs a separate iOS key).

> This is a different product from the **web** map. Web (`@vis.gl/react-google-maps`) uses
> the **Maps JavaScript API**; the native Android SDK uses **Maps SDK for Android** — same
> Google Cloud project, different API to enable, and a web key won't work as-is on Android.

## The Android key must live in the native manifest ⇒ dev build only

The Google key goes in `AndroidManifest.xml` as
`<meta-data android:name="com.google.android.geo.API_KEY" .../>`, compiled into the APK.
**Expo Go can't inject your key** (it's a fixed app with its own manifest) — it uses its
own bundled key, which currently **fails auth**, so the map is blank in Expo Go **no
matter what you configure**. Maps work **only in a dev build**. (Proven by reproducing the
same blank map + `host.exp.exponent` key in a clean Expo Go app.)

## Wiring the key (Expo)

Inject it from `.env` via a dynamic `app.config.ts` — keeps the key out of git:

```ts
// apps/mobile/app.config.ts
import type { ConfigContext, ExpoConfig } from "expo/config";
export default ({ config }: ConfigContext): ExpoConfig => ({
  ...config,
  android: {
    ...config.android,
    config: {
      ...config.android?.config,
      googleMaps: { apiKey: process.env.EXPO_PUBLIC_GOOGLE_MAPS_API_KEY },
    },
  },
});
```

```bash
# apps/mobile/.env
EXPO_PUBLIC_GOOGLE_MAPS_API_KEY=AIza...
```

Then **clean-prebuild + rebuild** (the key is baked into the manifest — a JS reload or an
incremental prebuild won't do it):

```bash
npx expo prebuild --clean && npx expo run:android
```

(`android/` is gitignored, so even the merged manifest with the key isn't committed.)

## Google Cloud checklist (the part that stays blank)

A dev build that presents your key but still shows a beige/blank map with only the
"Google" logo = **authorization failure**, not "key not found". Check, on the key's
project:

1. **"Maps SDK for Android" is ENABLED** — the #1 miss. A web key only has the JavaScript
   API. Enable at `console.cloud.google.com/apis/library/maps-android-backend.googleapis.com`.
2. **Billing is enabled** on the project (Google Maps requires a billing account;
   `INVALID_ARGUMENT` on "Error requesting API token" usually means billing or the API
   isn't enabled).
3. **Application restriction** is `None`, **or** "Android apps" with the correct package +
   SHA-1 (see below). A web key restricted to "HTTP referrers" will **never** work on Android.

Restriction changes are **server-side** → no rebuild, just wait ~2-5 min and reload the map.

## The correct SHA-1 (Expo signs with its OWN keystore)

Expo's prebuild generates `apps/mobile/android/app/debug.keystore` — **not** the system
`~/.android/debug.keystore`. Read the SHA-1 the app actually presents from the Expo one:

```bash
keytool -list -v -keystore apps/mobile/android/app/debug.keystore \
  -alias androiddebugkey -storepass android -keypass android | grep SHA1
```

Using the system keystore's SHA-1 (a common mistake) won't match — the map stays blank.

## Reading the real error (logcat)

```bash
adb logcat -d | grep -iE "Google Maps Android API|Authorization|API Key:|API key not found"
```

- `API key not found` → the `<meta-data>` isn't in the built APK (rebuild / clean prebuild).
- `Authorization failure` + `INVALID_ARGUMENT` + the presented `API Key` / `package;SHA-1`
  → the key exists but its project config (Maps SDK for Android / billing / restriction)
  doesn't match. The logged `Android Application (SHA;package)` is exactly what the app
  presents — restrict the key to *those* values, or set restriction to `None`.

## Framing the map

`initialRegion` is **sometimes ignored on Android's first render** (blank world view).
Recentre reliably with a ref on `onMapReady`:

```tsx
const mapRef = useRef<MapView>(null);
<MapView ref={mapRef} initialRegion={region}
  onMapReady={() => mapRef.current?.animateToRegion(region, 0)}>
```

## Related

- [Expo dev builds & Metro](./expo-dev-builds-and-metro.md) — the dev build the map needs.
