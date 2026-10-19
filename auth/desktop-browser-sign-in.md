# Playbook: Electron sign-in through the system browser

**The desktop app never shows a password field.** It opens the website's sign-in page in the
person's default browser; after they sign in there (any method the web offers, or "Continue as"
the account the browser already has), the browser returns to the app through a custom-scheme deep
link carrying a short-lived, single-use code bound to PKCE and state. The app exchanges the code for
a session of its own and stores it where it already stores sessions. When the link cannot reach the
app, the page shows the code to paste.

Built on Better Auth's official [`@better-auth/electron`](https://www.better-auth.com/docs/integrations/electron)
plugin. Last reviewed: 2026-10-19, against `better-auth` and `@better-auth/electron` 1.7.11.

## When to use, and why this shape

Use it when an Electron app needs accounts and any of these holds: OAuth providers (Google refuses
OAuth in embedded web views; [RFC 8252 §8.12](https://www.rfc-editor.org/rfc/rfc8252#section-8.12)
says native apps must not use embedded user agents for sign-in), payments that will live on the web,
or a wish to keep password managers and passkeys working.

| Alternative | Why not the default |
| --- | --- |
| In-app form + provider in a web view | Forbidden by Google and RFC 8252; hides the address bar; defeats password managers and passkeys |
| Loopback redirect (`http://127.0.0.1:<port>`) | Works unpackaged, but the browser ends on "you can close this tab" instead of returning; the plugin does not implement it |
| Device code only (RFC 8628) | No scheme to register, but typing a code is worse when the browser is on the same machine. Keep it as a possible later fallback |
| A hand-built code exchange | The security-critical part becomes yours to maintain, and the plugin replaces it at the next upgrade |
| The plugin's own cookie jar as the desktop session | Two stores of one session; every existing API caller would change how it authenticates |

## Prerequisites and order

1. **Better Auth 1.7.11** everywhere, with `@better-auth/electron` at the same version through the
   catalog — [upgrade playbook](./better-auth-1-7-upgrade.md), in its own PR.
2. **`bearer({ requireSignature: true })`** on the server. The exchange returns the signed session
   token in the `set-auth-token` header; the desktop sends it as `Authorization: Bearer`.
3. `verification` mapped in the adapter (the codes live there as `electron:<code>`).
4. A web app on the same origin as the auth proxy that can host two new pages.
5. A shared package all three apps can value-import for the constants (a `domain` constants module).

PR split that worked: (1) upgrade → (2) server plugin + web pages, testable with `curl` alone →
(3) desktop. Retarget (3) to `main` once (2) merges (see the [README](./README.md#process-lessons-from-the-reference-implementation)).

## The flow

```
desktop main                       browser (<web-origin>)                         API (/api/auth)
verifier, state; challenge = S256(verifier)
open /auth/desktop?client_id=<id>&state=…&code_challenge=…[&intent=sign-up]
                                   ├ signed in  → "Sign in to <App> on this computer?"
                                   │   Continue ─────── POST /electron/transfer-user?<query> ─► mint code
                                   │   Switch account → sign out, same URL shows the form
                                   └ signed out → sign-in / sign-up / Google, query forwarded
                                       POST /sign-in/email?<query> (or the OAuth round trip) ─► new session
                                                                                                + mint code
                                   ◄─ Set-Cookie: <prefix>.<client id>=<base64url {identifier,state}>
                                      (readable by JS, short-lived)
                                   /auth/desktop/done → read cookie → ensureElectronRedirect()
◄── <scheme>:/auth/callback#token=<cookie value>     (or the person pastes it)
POST /electron/token {token, state, code_verifier}, Origin: <scheme>:/ ─────────────────────► consume + check
◄──────────────────────────────── 200 {token, user} + set-auth-token: <signed session token>
commit the signed token to the existing encrypted session store
```

"Mint code": 32 random characters stored as `electron:<code>` with `{ userId, codeChallenge, state }`,
expiring after 300 s, deleted when read.

## Steps

### 1. Shared constants (`packages/domain/src/constants/desktop-auth.ts`)

`DESKTOP_AUTH_CLIENT_ID` (a public id, not a secret), `DESKTOP_AUTH_SCHEME`,
`DESKTOP_AUTH_CALLBACK_PATH` (`/auth/callback`), `DESKTOP_AUTH_ORIGIN` (`` `${scheme}:/` ``) and
`DESKTOP_AUTH_COOKIE_PREFIX` (Better Auth's default `better-auth`). Server, web and desktop main read
the same values, so they cannot drift. Desktop **preload** code must not value-import a workspace
package if the preload bundle externalizes them — keep these in main.

Scheme name: the readable `<app>` rather than the reverse-domain form RFC 8252 §7.1 suggests is
acceptable because PKCE makes an intercepted code useless, which is the risk the reverse-domain form
reduces. Reserve other paths now (for example `<scheme>://billing/refresh`, which must only ever ask
the app to refresh, never grant anything).

### 2. Server (`apps/server`)

- `plugins: [..., electron({ clientID: DESKTOP_AUTH_CLIENT_ID })]`. It adds hooks on `/sign-in/*`,
  `/sign-up/*` and `/callback/*` that mint a code when the request carries the desktop query, plus
  `POST /electron/transfer-user`, `POST /electron/token` and `GET /electron/init-oauth-proxy` (the
  last unused by a browser-driven flow).
- `trustedOrigins`: add `DESKTOP_AUTH_ORIGIN` (`<scheme>:/`). Better Auth 1.7 matches a host-bearing
  pattern against the host exactly, so `<scheme>://app` would **not** match the `Origin: <scheme>:/`
  the Electron client sends. Keep any literal older desktop builds still send until no supported
  build sends it.
- Nothing else: the exchange creates a **new session for the device**, with its own user agent and
  IP, so a later "your devices" page can list and revoke them with Better Auth's session endpoints.

### 3. Web (`apps/web`)

- `lib/auth/desktop-auth-client.ts`: `createAuthClient({ plugins: [electronProxyClient({ protocol,
  clientID, cookiePrefix, callbackPath })] })`. Only the desktop pages import it, so it ships in its
  own chunk and public pages never load it.
- `lib/desktop-auth.ts` (pure): parse the query — `client_id` equal to the constant, `state` a short
  base64url-ish string, `code_challenge` exactly 43 base64url characters (an S256 challenge); null on
  anything else. Helpers for the done path and the deep-link URL in the exact form
  `ensureElectronRedirect` builds.
- `/auth/desktop`: invalid or missing query → "Open sign-in from the <App> app" (never sign the
  browser in and stop). Signed in → **Continue** (`electron.transferUser`) or **Switch account**
  (sign out, same URL). Signed out → the regular sign-in form, or sign-up with `intent=sign-up`, both
  forwarding the query with `fetchOptions: { query }`; on success navigate to the done page, not to
  the post-sign-in landing. The Google button sends the same `fetchOptions`, `callbackURL` = the done
  page with the query and `errorCallbackURL` = `/auth/desktop` with the query; the plugin keeps the
  query across the OAuth round trip in a signed cookie and mints on `/callback/google`.
- `/auth/desktop/done`: poll `electron.getAuthorizationCode()` (100 ms, up to 10 s). When it appears,
  **keep the value**, call `ensureElectronRedirect()`, and show "Returning to <App>…". After about
  4 s show the fallback: **Open <App>** (the same deep link again) and **Copy code**. No cookie after
  10 s → "No sign-in code here; start again from the app".
- All `/auth/desktop*` routes `noindex`; their copy loads only on those routes.

### 4. Desktop main (`apps/desktop/src/main`)

- `services/browser-sign-in.ts`: one `createAuthClient({ baseURL: <api>, plugins: [electronClient({
  signInURL, protocol: { scheme }, clientID, cookiePrefix, storage })] })` per intent (the plugin
  appends its query to a fixed `signInURL`, so sign-up carries `intent=sign-up` in its own URL).
  - `begin(intent)` → `requestAuth()` (makes verifier and state, keeps them, opens the browser).
  - `exchange(code)` → check the code decodes as base64url JSON `{ identifier, state }` (else
    "invalid code" without a request) → `authenticate({ token: code, fetchOptions: { onSuccess } })`,
    capturing `set-auth-token` (fallback: the session token from `Set-Cookie`) → return token + user.
  - `storage`: a **synchronous in-memory** map (the interface is sync; a returned Promise reads as
    empty), cleared in `finally` after every exchange. One persistent copy of the session exists: yours.
  - Map errors: "verifier not found" → this sign-in was not started by this run; `TypeError` →
    network; 4xx → invalid or expired code; anything else → unknown.
- Commit the token through the **existing** session store (encrypted with `safeStorage`; refuse to
  write when encryption is unavailable). Signing in over an existing session signs the old one out
  on the server. Sessions stored before the change keep working.
- `deep-link-router.ts` (pure, testable with strings): `receive(url)`, `receiveArgv(argv)`, `start()`.
  Queue links until the app is ready, then dispatch: `auth/callback#token=…` → focus + exchange;
  any other path of the scheme → deliberately inert. **Never log the URL** — it carries a code.
  Accept both `<scheme>:/auth/callback` and `<scheme>://auth/callback` (the second parses `auth` as
  the host).
- `index.ts`, before `app.whenReady`:
  - **one** `app.requestSingleInstanceLock()`; quit when not acquired;
  - `if (app.isPackaged) app.setAsDefaultProtocolClient(scheme)`;
  - `app.on("open-url", (e, url) => { e.preventDefault(); router.receive(url) })` — macOS, including
    a cold start by link, which fires before `ready`;
  - `router.receiveArgv(process.argv)` — Windows/Linux cold start;
  - in the existing `second-instance` handler, `router.receiveArgv(argv)` before your window logic;
  - inside `whenReady`, build the sign-in client and the store, then `router.start()`.
- **Do not call the plugin's `setupMain`.** It takes the single-instance lock again, registers its own
  `second-instance` handler (which focuses the window regardless of app state — for example while a
  recording hides it), and installs CSP rewriting, IPC bridges and a custom protocol you may not
  want. **Do not use `setupRenderer`** either: the renderer must never receive the token; keep your
  own typed IPC (begin, paste code, cancel, and a status broadcast — names are yours).
- Remove the password IPC and its main-process code in the same PR.

### 5. Registration and environment

- `electron-builder.yml`: `protocols: [{ name: <App>, schemes: [<scheme>] }]` (Info.plist on macOS,
  registry on Windows, desktop file on Linux).
- `setAsDefaultProtocolClient` only when packaged: a dev run is Electron's own binary; registering it
  would, on Windows, take over every `<scheme>://` link from the installed app, and macOS routes
  custom schemes only to packaged apps anyway. **Development uses the paste code.**
- Build-time variable for the web origin next to the API URL (with electron-vite, `MAIN_VITE_WEB_URL`).
  The release workflow fails with a clear error when it is unset; locally the app still starts and
  "Sign in" reports "this build does not know where to sign in". CI E2E sets a local value inline.

### 6. Renderer

"Sign in with your browser" and "Create account" (same flow, sign-up intent). While waiting: **Open
browser again**, **Cancel**, and a **Paste code** field. One message per error kind. No password field
anywhere — an E2E test asserts it.

## The paste-code fallback — what is actually copied

The desktop's `authenticate({ token })` base64url-decodes its input into `{ identifier, state }` and
finds the verifier by that state. So **Copy code copies the readable cookie's value, not the raw
32-character code**. The cookie is named `<cookiePrefix>.<clientID>` (with a custom client id, *not*
`better-auth.electron`), and `ensureElectronRedirect()` deletes it as it redirects — read it first.
The desktop drops its verifier on the first exchange attempt, even a failed one, so after a failed
paste the person starts again from the app; the page and the app both say so.

## Tests that must exist

Server, in process (real Better Auth, memory adapter, origin check forced on, fake token endpoint):

| Case | Protects against |
| --- | --- |
| Web sign-in with the query → exchange → new session; the signed `set-auth-token` is accepted as Bearer | The happy path |
| Signed-in visitor → `transfer-user` → exchange | "Continue as" |
| Google round trip with the query → still a 302 to the done page, code minted | OAuth path losing the query |
| Sign-up mints a code too | New users stuck in the browser |
| Wrong verifier rejected **and** the code burned | Brute force on the verifier |
| Reused code rejected; 301 s rejected, 290 s accepted; wrong state rejected | Replay, expiry, CSRF |
| Another `client_id` mints nothing | Foreign clients |
| `<scheme>:/` trusted, an unknown origin refused, a host-bearing pattern alone not enough | Origin regressions |

Web: query parsing, the real proxy client's cookie name, the sign-in page states (invalid, confirm,
Continue, transfer failure, Switch account, sign-up intent, OAuth error), the done page (redirect,
late cookie, fallback copies the cookie value, no code). Desktop: link parsing (both slash forms,
Windows argv with leading flags, cold start), the router (queue until start, never logs), exactly one
single-instance lock and no `setupMain`, the client wrapper (token capture, error mapping, jar
cleared), the store (commit, legacy session read, refusal without encryption), the renderer states.

## Pitfalls hit

- The trusted-origin pattern: `<scheme>://app` looked right and did not match.
- The cookie name derived from the client id, and the copy-the-cookie-not-the-code rule (the spec
  had it backwards; the web PR caught it with a test that runs the real proxy client).
- Two `@better-auth/core` copies from a `jose` version split broke plugin types.
- Chrome accepts the plugin's `Secure` cookie on `http://localhost`; Safari may not. Test dev in Chrome.
- A legacy `Origin` literal was still sent by another desktop call (resend verification), so it could
  not be dropped with the password form.
- Two PRs of the stack merged into their base branches after those had landed; a carrier PR brought
  them to `main`.
- Not verified by tests alone: the cookie through the web proxy on a real domain, the deep link
  into a packaged app on macOS and Windows, cold start by link. Do those by hand once per platform.

## How to test it yourself

Server and web alone, before the desktop exists:

1. Start the stack; open, in Chrome,
   `http://localhost:<web-port>/auth/desktop?client_id=<id>&state=test&code_challenge=<S256 of a verifier you chose>`
   (compute the challenge with `node -e` — base64url of the SHA-256 of the verifier).
2. Sign in. On the done page wait for **Copy code**; decode it
   (`Buffer.from(code, "base64url").toString()` → `{"identifier":…,"state":"test"}`).
3. `curl -i -X POST http://localhost:<web-port>/api/auth/electron/token -H 'content-type: application/json' -H 'origin: <scheme>:/' -d '{"token":"<identifier>","state":"test","code_verifier":"<verifier>"}'`
   → `200` with `set-auth-token`. Use it as `Authorization: Bearer` on `/api/auth/get-session`.
4. Repeat step 3 → `404` (single use). A wrong verifier → `400`.
5. Open the URL again while signed in → "Continue as…"; add `&intent=sign-up` signed out → sign-up form;
   open `/auth/desktop` with no query → the explanation.

With the desktop:

6. Dev build: **Sign in with your browser** → sign in → **Copy code** → paste in the app → signed in.
   Quit and relaunch: still signed in. Sign out: only the desktop session ends.
7. Paste a used code → error asking to start again.
8. Packaged build: the browser asks to open the app and the app signs in by itself. Quit the app and
   open a stale link → "This sign-in was not started from this app".

## Reference implementation

Kaipu Record, owner's product: [ADR 0025](https://github.com/niway-dev/kaipu-record/blob/main/apps/documentation/src/content/docs/architecture/decisions/0025-desktop-signs-in-through-the-browser.mdx)
and [the design spec](https://github.com/niway-dev/kaipu-record/blob/main/apps/documentation/src/content/docs/specs/2026-10-19-desktop-browser-sign-in-design.md);
[#330](https://github.com/niway-dev/kaipu-record/pull/330) (upgrade),
[#333](https://github.com/niway-dev/kaipu-record/pull/333) (server plugin, web pages, server flow test),
[#332](https://github.com/niway-dev/kaipu-record/pull/332) (desktop: router, store, renderer),
[#335](https://github.com/niway-dev/kaipu-record/pull/335) (the carrier PR that landed #333 and #332 on `main`),
[#340](https://github.com/niway-dev/kaipu-record/pull/340) (done page redesign; sign-up returns at once).

## Related

- [better-auth-1-7-upgrade.md](./better-auth-1-7-upgrade.md) — the required version, and the origin change
- [google-sign-in.md](./google-sign-in.md) — the button the desktop page reuses
- [web-sign-in-flows.md](./web-sign-in-flows.md) — sign-up from the desktop returns before verification
- [desktop/main-process-architecture.md](../desktop/main-process-architecture.md) — where the router and the lock live
- [desktop/ipc-contract.md](../desktop/ipc-contract.md) — the typed bridge the renderer uses instead of the plugin's
