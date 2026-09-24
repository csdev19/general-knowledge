# Client State Persistence: Where the Source of Truth Lives

_Deciding where a client app's persisted state belongs when the app has more than one runtime — Electron main vs renderer, React Native JS vs native. Why `persist` middleware is the web's right answer and the wrong one here, and the rule that replaces it._

---

## The trap

You need a setting to survive a restart. You reach for the store you already
have, and it has an answer ready:

```ts
// zustand; redux-persist, pinia-plugin-persistedstate and friends are the same shape
export const useSettings = create(
  persist((set) => ({ engine: "whisper", setEngine: (engine) => set({ engine }) }), {
    name: "settings",
  }),
);
```

One line, state survives reloads, done. In a browser SPA this is correct and
you should stop reading.

It is correct there for a reason worth naming, because the reason is exactly
what stops being true everywhere else: **a browser page is one runtime.** The
JavaScript that writes the value is the same JavaScript that reads it, and
`localStorage` is the only durable store it has. There is nobody else to
disagree with.

Desktop and mobile apps are not one runtime. That single fact is the whole
subject of this document.

---

## The rule

> **Persisted state belongs to the process that _acts_ on it, not the one that
> displays it.**

To apply it, ask one question about each piece of state:

**Who reads this value in order to _do_ something?**

| Who acts on it                                             | Where it lives                             |
| ---------------------------------------------------------- | ------------------------------------------ |
| Only the UI that renders it                                | Client-local storage. `persist` is fine.   |
| A different process (a native layer, a privileged backend) | That process owns it; the UI mirrors it.   |
| Nothing — there is no UI running at the time               | Definitely not the UI.                     |

And one sharper test that settles the ambiguous cases:

> **Does anything need this value when no UI exists?**

Launch-time work, background jobs, a notification handler, a widget, a
scheduled task, a queued retry. If any of them needs the value, the UI cannot
be its home — not because it would be untidy, but because at the moment the
value is needed there is no UI to read it from.

---

## Why multi-runtime clients break the web's default

### Electron

Two processes, and the split is not cosmetic:

| | Main (Node) | Renderer (Chromium) |
| --- | --- | --- |
| Spawns child processes, opens sockets | ✅ | ❌ |
| Reads/writes the filesystem | ✅ | ❌ (with `contextIsolation`) |
| Holds credentials (`safeStorage`) | ✅ | ❌ |
| Runs before any window exists | ✅ | — |
| Runs after every window closes | ✅ | — |
| Has `localStorage` | ❌ | ✅ |

The last row is the problem. `localStorage` exists only in the process that
can act on almost none of it. A preference kept there is invisible to the
process that actually uses it.

The failure is not subtle when it happens:

> A desktop app let people choose between two speech-recognition engines. The
> choice lived in the renderer. Main — the process that actually spawns the
> engine — kept reading its own stored default. The dropdown said one engine;
> the app ran the other. Nobody noticed for two days, because both produce
> plausible text. The bug was only provable from a log line that named the
> engine that had really started.
>
> The same app ran its post-session summary from `bootstrap()`, before any
> window was created. Had the preference lived in the renderer, main would
> have had to open a hidden window to read its own configuration.

The second half is the general point: **a process that has to start a UI to
read its own configuration has the architecture inverted.**

### React Native / Expo

The boundary is softer — MMKV and AsyncStorage are backed by native storage,
so native code _can_ read them — but it is still there, and it appears in the
places that matter most:

- **Widgets** (iOS WidgetKit, Android Glance) render with your app not
  running. They cannot call into your JS store.
- **Notification handlers** and **background tasks** (headless JS, BGTask,
  WorkManager) may run in a stripped-down context, or none.
- **Share / action extensions** are separate processes with separate sandboxes.
- **Native modules** reading a JS-owned store is a layering inversion even
  when it is technically possible.

So on mobile the rule produces a three-way split:

| Kind of state                                        | Where                                                              |
| ---------------------------------------------------- | ------------------------------------------------------------------ |
| Pure UI (collapsed section, last tab, unsent draft)  | JS store + `persist` over MMKV/AsyncStorage                        |
| Anything a widget/extension/background task reads    | Shared native container — App Group + `UserDefaults` (iOS), DataStore/SharedPreferences (Android); MMKV configured with the app-group id |
| Tokens, keys, anything secret                        | Keychain / Keystore (`expo-secure-store`) — **never** AsyncStorage |

`expo-secure-store` is not "persistence with extra steps": AsyncStorage and
`localStorage` are plaintext on disk. A token in either is a token on the
device's filesystem.

### And one that is not client state at all

State the server owns is a **cache**, not a preference. It belongs in a query
cache with its own persistence and invalidation (TanStack Query's persister,
RTK Query, Convex's own client), never hand-rolled into a preference store.
The giveaway: if the value can change while the app is closed and you would
want the new one, it is a cache.

---

## The shape of the right answer

Two **layers**, which is not the same as two sources of truth:

```
┌──────────────────────────────────────────┐
│ OWNER — persists and validates           │
│   Electron: main + a JSON file in        │
│   userData. Mobile: native/shared store. │
└──────────────────────────────────────────┘
        ▲                    │
        │ setter (IPC)       │ broadcast on change
        │                    ▼
┌──────────────────────────────────────────┐
│ MIRROR — the UI's in-memory copy         │
│   Seeded at startup. Never writes to     │
│   disk itself. Re-render only.           │
└──────────────────────────────────────────┘
```

Three properties make this work, and each one fails loudly if you drop it:

**1. The setter _is_ the write.** No `persist` middleware on the mirror. The
mirror has nothing of its own to save — it writes through, and the answer is
what it then holds.

**2. The UI renders what came back, never the optimistic patch.** The owner
validates: it may clamp, reject, or substitute. Rendering what you _asked_ for
lets a refused value sit on screen looking accepted.

```ts
// The whole mirror, in one function.
const write = async (patch: Partial<Settings>): Promise<void> => {
  set({ settings: await ipc.settings.update(patch) }); // ← what main decided, not `patch`
};
```

**3. Seed before first paint, not in an effect.** Settings decide language,
theme and locale; reading them after mount means rendering the wrong ones for
a frame. Seeding is a plain function call before the tree mounts — not a
provider, not a hook, because nothing _renders_ a seed.

```ts
const settings = await ipc.settings.get();
hydrate(settings); // plain call
subscribe(); // follow the owner's broadcasts
createRoot(el).render(<App />);
```

An unseeded mirror should **throw**, not fall back to a plausible default.
Rendering somebody's settings as `en` / `system` / the first engine in a list —
when they are none of those — is a lie no test will catch.

### Where zustand still earns its place

All of the above is an argument against one middleware, not against the
library. As the mirror, a store beats a React context on every axis that
matters here:

- No provider in the tree, so **non-component code can read and write it** —
  which is what makes "seed before first paint" a plain call.
- Selector subscriptions instead of a context value that re-renders every
  consumer on every change.
- No `useMemo` with a dependency array as long as your settings object.

Use the store. Skip the middleware whose job another process is already doing.

---

## Checklist

For each piece of persisted state:

- [ ] Named the process that **acts** on it — not the one that renders it.
- [ ] Asked whether anything needs it **when no UI is running**.
- [ ] Exactly **one writer**. A value written by two processes has no truth.
- [ ] Secrets in Keychain/Keystore/`safeStorage`, never plaintext KV.
- [ ] Server-owned data in a **query cache**, not a preference store.
- [ ] The UI stores **what came back** from the owner, not what it sent.
- [ ] The mirror is seeded **before first paint**, and throws if read unseeded.
- [ ] Defaults name something a **fresh install can actually reach**. A
      default pointing at a provider that needs a credential no new install
      has is a guaranteed first-run failure.

---

## Anti-patterns

| Anti-pattern                                           | Why it bites                                                                                 |
| ------------------------------------------------------ | -------------------------------------------------------------------------------------------- |
| `persist` middleware over a store that already writes through IPC | Two copies, diverging silently. The UI shows one thing, the acting process does another.      |
| Reading a UI-owned preference to configure a native subsystem at launch | The subsystem starts before the UI, so it reads a stale value or none.                        |
| Two processes writing the same key                     | Last-write-wins across processes with no ordering guarantee. Unreproducible bugs.              |
| A rehydrated store treated as authoritative            | Rehydration is async; the first frames run on defaults. Anything that fires there uses them.   |
| Defaults chosen for the happy path, not the fresh install | Every unattended job resolving against that default fails until somebody notices.              |
| A per-screen "draft" the user expects to be remembered | People do not find the second settings screen. What they picked in the flow is what they meant. |

That last row is worth its own sentence, because it is a product bug wearing
architecture clothes. If someone re-makes the same choice before every run,
the choice was never a draft. Persist it — and note that when the drafted
value and the stored value differ, every unattended job reads the **stored**
one, so the gap between them is not cosmetic.

---

## Related

- [`../desktop/main-process-architecture.md`](../desktop/main-process-architecture.md) — what main owns and why.
- [`../desktop/ipc-contract.md`](../desktop/ipc-contract.md) — the typed bridge the mirror's setters cross.
- [`../mobile/mobile-app.md`](../mobile/mobile-app.md) — Expo/RN app structure.
- [`./security-hardening.md`](./security-hardening.md) — where secrets belong.
