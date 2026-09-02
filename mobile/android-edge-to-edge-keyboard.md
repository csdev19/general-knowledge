# Android 15+ Edge-to-Edge — `KeyboardAvoidingView` Collapses Into a Grey Curtain (device-only)

_Tapping a `TextInput` on a real Android 15/16 phone paints a grey area and the keyboard
appears to never open; navigating to another screen shows the same grey and looks like
"nothing happened". **Metro prints no error.** The same JS bundle behaves perfectly on the
Android emulator. Root cause: since API 35 the IME no longer resizes the window, so
`KeyboardAvoidingView` with `behavior="height"` measures a stale viewport and collapses its
child to zero — revealing `react-native-screens`' default light-grey background._

> **Search keywords:** grey screen Android TextInput · keyboard does not open Expo ·
> KeyboardAvoidingView broken Android 15 · edge-to-edge Android 16 keyboard · grey overlay
> tapping input · works on emulator fails on device · Pixel 9 keyboard React Native ·
> `behavior="height"` Android · window not resized IME · adjustResize API 35 ·
> react-native-screens grey background · expo-router Link does nothing.

## Symptom

- Real device (Pixel 9 Pro, Android 16 / API 36): tap the email field → a grey "curtain",
  no usable keyboard. Tap a `<Link>` to another screen → same grey, appears to do nothing.
- Android emulator (API ≤ 34): everything works.
- **Metro shows no exception.** There is no crash — it is a layout miscalculation.

The two symptoms look unrelated (keyboard vs navigation) which sends you hunting for a
navigation bug. They are the same bug: the destination screen used `behavior="height"` too,
so it also rendered collapsed. You *did* navigate — you just arrived at a broken screen.

## Root cause

Two independent facts that only bite together:

**1. The IME no longer resizes the window (API 35+).** `adjustResize` used to shrink the
window when the keyboard opened. Under edge-to-edge it does not: the window stays full
height and the IME is reported as an *inset* instead. React Native's `KeyboardAvoidingView`
with `behavior="height"` is built on the old resize behaviour, so it measures an unchanged
viewport and collapses its child.

- [facebook/react-native#49759](https://github.com/facebook/react-native/issues/49759) —
  closed. Reproduced **on a Pixel 9**. Works at `targetSdkVersion=34`, breaks at 35.
- [react-native-keyboard-controller#861](https://github.com/kirillzyusko/react-native-keyboard-controller/issues/861)
  — closed "not planned"; the maintainer confirms core RN has the same bug. It is a
  platform regression, not a library bug.

**2. `react-native-screens` paints a light-grey default background on Android.** Wherever a
screen does not cover the window — e.g. because the layout just collapsed — that grey shows
through. Containers styled `{ flex: 1 }` with no `backgroundColor` have nothing of their own
to paint.

**Why the emulator lies:** Android 15 (API 35) made edge-to-edge the default but still
allowed opting out via `windowOptOutEdgeToEdgeEnforcement`. **Android 16 (API 36) removed
that escape hatch.** An emulator on API ≤ 34 keeps the old resize behaviour, so the bug
cannot reproduce there. The emulator is not lying — it is a different Android.

## Fix

On Android, `behavior` must be `undefined`. This is what
[Expo's keyboard-handling guide](https://docs.expo.dev/guides/keyboard-handling/)
prescribes — neither `"height"` nor `"padding"`.

```tsx
<KeyboardAvoidingView
  // Android must stay `undefined`: since API 35 the IME no longer resizes the window
  // (edge-to-edge is mandatory from Android 16), so "height" measures a stale viewport
  // and collapses the layout — facebook/react-native#49759.
  behavior={Platform.OS === "ios" ? "padding" : undefined}
  style={{ flex: 1 }}
>
```

And give screens a real background, anchored to the theme so dark mode stays correct:

```tsx
// app/_layout.tsx
const theme = colorScheme === "dark" ? DarkTheme : DefaultTheme;

<Stack screenOptions={{ contentStyle: { backgroundColor: theme.colors.background } }}>
```

The background alone is **not** a fix — it turns a grey broken screen into a white broken
screen. Fix the `behavior` first; the background stops the failure from being disguised as
a mysterious "curtain" next time.

For anything beyond a simple centred form, use
[`react-native-keyboard-controller`](https://github.com/kirillzyusko/react-native-keyboard-controller),
which reads IME insets via `WindowInsets` instead of depending on window resize. **It is not
in Expo Go** — it needs a dev build.

Also check `app.json`: `android.softwareKeyboardLayoutMode` should be `"resize"` (the
default). `"pan"` is worse under edge-to-edge.

## How to detect it in 30 seconds

The decisive measurement is whether the window resizes when the IME opens. Drop this in the
screen and read Metro:

```ts
const win = Dimensions.get("window");
Keyboard.addListener("keyboardDidShow", (e) => {
  const after = Dimensions.get("window");
  console.log(`ime=${e.endCoordinates.height} window=${after.height} (was ${win.height})`);
  if (after.height === win.height) {
    console.log("WARN window did not resize → KeyboardAvoidingView behavior='height' cannot work");
  }
});
```

`keyboardDidShow` firing with a real height **while the window height is unchanged** is the
signature. It also proves the keyboard *is* opening — which kills the tempting theories
(autofill overlay, a native view swallowing touches, a broken auth call).

Log `Platform.Version` too: it is the API level, and comparing device vs emulator explains
the whole discrepancy at a glance.

## Debugging lessons from this one

**Instrument the layers separately, and don't put reactive values in a "did I mount?"
effect.** A first attempt logged mount state from a `useEffect` whose deps included
`insets.*`. Opening the keyboard changed the insets, the effect re-ran, and the log looked
exactly like a remount — sending the investigation after a phantom navigation loop that did
not exist. Use `[]` deps for mount/unmount, and a separate effect for anything reactive.

**Order the questions so each answer eliminates a layer.** Does the log pipe work at all →
does the touch reach JS (`onFocus`) → does the IME open (`keyboardDidShow`) → does the
window respond (`Dimensions`). Each step kills a class of hypotheses. Here, `onFocus`
firing eliminated "a native view is intercepting touches", and `keyboardDidShow` eliminated
"the keyboard never opens" — which was the user-visible symptom and the wrong description of
the bug.

**Two symptoms that look unrelated are worth a shared-cause check before you split the
investigation.** Grep for the suspect pattern across the codebase: here, only the two broken
screens used `behavior="height"`; three other screens already used `undefined` and worked
fine. That correlation was stronger evidence than any of the reasoning that preceded it.

## Prevention

- **`behavior="height"` on Android is always wrong now.** Grep for it:
  `grep -rn 'behavior=' app/ components/`. Every occurrence should be
  `Platform.OS === "ios" ? "padding" : undefined`.
- **Always set an explicit screen background** on Android via `contentStyle`. Grey is the
  default, and it reads as "something broke" rather than "no background set".
- **Test on a real device with a current Android before shipping.** Emulators commonly run
  older API levels, and edge-to-edge enforcement is exactly the class of change they hide.
  See [expo-dev-builds-and-metro.md](./expo-dev-builds-and-metro.md) for the dev-build path —
  and note that Expo Go, with its own manifest and theme, is not representative of your app
  either.

## References

- [facebook/react-native#49759 — KeyboardAvoidingView does not keep input in view under Android 15](https://github.com/facebook/react-native/issues/49759)
- [Expo — Keyboard handling guide](https://docs.expo.dev/guides/keyboard-handling/)
- [react-native-keyboard-controller#861](https://github.com/kirillzyusko/react-native-keyboard-controller/issues/861)
- [react-navigation#12401 — Gaps between animating screens (New Architecture)](https://github.com/react-navigation/react-navigation/issues/12401)
- [software-mansion/react-native-screens#3483 — Android transition glitches](https://github.com/software-mansion/react-native-screens/issues/3483)
