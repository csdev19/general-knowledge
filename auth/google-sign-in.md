# Playbook: Google sign-in with Better Auth

**Adding the Google provider is ten lines; deciding what happens when a Google email matches an
existing account is the actual work.** Decide the linking policy first, make the destructive part
fail closed, and only then wire the button. Everything else — the Cloud Console, the consent screen,
the legal pages and the secrets — is a checklist this page gives you in order.

Last reviewed: 2026-10-19, against `better-auth` 1.7.11.

## When to use

You have Better Auth on a server behind a web app and want "Continue with Google" on the web (and,
through the [desktop browser flow](./desktop-browser-sign-in.md), on the desktop). Apple and GitHub
follow the same shape; the linking section applies to every provider that reports a verified email.

## Prerequisites and order

1. **Better Auth ≥ 1.7.7** — see [the upgrade playbook](./better-auth-1-7-upgrade.md). The
   OAuth-state advisory fixed in 1.7.7 is exactly the surface you are about to open.
2. **`verification` mapped in the adapter.** OAuth state is stored there by default whenever a
   database is configured (`storeStateStrategy: "database"`); without the mapping every callback
   fails.
3. **Public privacy policy and terms pages.** The consent screen's branding page asks for both URLs
   ([legal templates](../legal/README.md)).
4. A transactional email sender if you keep email+password (the guard below sends nothing, but the
   accounts it protects are the unverified ones).

Suggested PR split: (a) config — env schema, provider builder, explicit base URL, deploy workflow,
runbook; inert until secrets exist. (b) Linking guard, button, account page controls. (c) Legal
copy for Google. (b) and (c) can go in parallel; the owner's console work happens between (a) and (b).

## Decide the linking policy first

Better Auth links an OAuth sign-in to an existing user with the same email when the provider says
the email is verified. That is the right default — and a takeover path when your app also has an
**unverified** way to claim an email:

1. Someone signs up as `ana@example.com` with a password. Nobody checks the inbox.
2. Ana later uses "Continue with Google". Google says `ana@example.com` is verified; Better Auth links.
3. The first registrant still has a password into Ana's account.

Pick one answer, write it down (it is ADR material), and test it:

| Option | What you do | Cost | Fits when |
| --- | --- | --- | --- |
| **A. No unverified claim exists** | Turn email+password off; sign-in only by OAuth or by an emailed link/code (which proves the inbox) | No passwords | The product never needed passwords |
| **B. Verify before password sign-in** | `requireEmailVerification: true` on email+password | Every password user must click a link before first sign-in | Passwords are secondary |
| **C. Pre-hijack guard** | On an implicit Google link to an **unverified** password user, delete that password and every earlier session *before* the Google row is written | A hook you own, and its tests | You already have password users who sign in unverified |

Since 1.7, Better Auth refuses option C's link by default (`requireLocalEmailVerified: true` →
`account_not_linked`). Keep that default for A and B. Set it to `false` only together with the guard.

### The pre-hijack guard

Put every decision in a **pure function** and keep the hook thin:

```ts
// apps/server/src/lib/account-linking.ts — shape, not a drop-in
decideLinkSideEffects({ providerId, flow, user, existingAccounts, google }) →
  | { kind: "none" }
  | { kind: "refuse"; reason }                 // Google email unverified or different
  | { kind: "drop-unverified-credential"; credentialAccountIds }
  | { kind: "mark-email-verified" }            // manual link, same verified email
```

Rules that matter:

- **Destructive work in `databaseHooks.account.create.before`, never in `after`.** Without database
  transactions (the Drizzle adapter defaults to none; `neon-http` has none), a failure in `after`
  leaves the Google row written. The next sign-in takes the "already linked" branch, which runs no
  create hook, marks the user verified, and leaves the attacker's password in place. In `before`, a
  throw means nothing was linked.
- **Abort with a plain `Error`.** Better Auth answers it with `?error=unable_to_link_account`. An
  `APIError` without a `code` escapes the callback as a bare 403 (1.7).
- **Delete sessions with `internalAdapter.deleteUserSessions(userId)`.** Inside the hook the new
  session does not exist yet (the order is link → `emailVerified: true` → `createSession`), so it is
  spared. Desktop bearer sessions are rows too and die with it.
- **Tell a manual link from a sign-in by the OAuth state, not by path.** `getOAuthState()` (from
  `better-auth/api`) returns the parsed state; its `link.userId` is set only when a signed-in user
  started `linkSocial`. Treat anything unclassifiable as an implicit sign-in: the implicit rules
  only ever remove access.
- **Do not filter the hook by path.** `POST /sign-in/social` with an `idToken` also runs implicit
  linking at a different path. Close that door instead: `disableIdTokenSignIn: true` on the provider
  if you only use the redirect flow.
- **Read `email` and `email_verified` from the `id_token` Better Auth just received** (base64url
  payload, no signature check needed: it came from Google's token endpoint over TLS in this request).
  Missing or `false` claims refuse the join.
- Leave a one-time notice (a `verification` row, consumed with `consumeVerificationValue`) so the
  dashboard can say "we removed an old password from your account" once.

Other linking options: keep Google **out** of `trustedProviders` (its `email_verified` must be
required), `allowDifferentEmails: true` only matters for `linkSocial` by a signed-in user, and name
and image are never overwritten on link (Better Auth default — assert it).

## Steps

### 1. Server config (`apps/server`)

- Env schema: optional `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `BETTER_AUTH_URL` (a URL, no
  trailing slash). Empty counts as absent.
- A pure `buildSocialProviders(env)` that registers Google **only when both values exist**, so the
  API boots in dev and CI without them, plus one warning per isolate when it is off. Options:
  `prompt: "select_account"` (people with work and personal accounts choose consciously),
  `disableIdTokenSignIn: true`, default scopes `openid email profile` (all non-sensitive).
- **`baseURL: env.BETTER_AUTH_URL`, explicitly, set to the web origin.** Better Auth builds the
  `redirect_uri` from it, and the OAuth state cookie is set on the origin the browser is on. Without
  it, Better Auth falls back to the request origin, which behind the web proxy is the API — the
  callback then arrives without the state cookie and fails with a state mismatch. On Workers do not
  rely on Better Auth reading `BETTER_AUTH_URL` from `process.env`; pass it.
- `account: { accountLinking: {...}, encryptOAuthTokens: true }`. Encrypt the stored access and refresh
  tokens even if you never call Google APIs; the `id_token` is stored as received.
- `databaseHooks: { account: { create: { before, after } } }` from the guard module (option C).
- Unique index on `account (provider_id, account_id)`: two parallel callbacks cannot link the same
  Google identity twice.
- If an API middleware gates `/api/v1`, read sessions with `disableCookieCache: true` there, so a
  session revoked by the guard stops working at once. Server-rendered pages can keep the cookie cache
  (a revoked browser may still *render* for up to its `maxAge`, but cannot call the API).

### 2. Web (`apps/web`)

- One Google button component used on sign-in, sign-up, the desktop sign-in page and the account
  page. `authClient.signIn.social({ provider: "google", callbackURL, errorCallbackURL })`.
  `callbackURL` comes from the [post-sign-in resolver](./web-sign-in-flows.md#6-where-every-sign-in-lands);
  `errorCallbackURL` is the page the button lives on.
- **No feature flag on the button.** The server's env already decides whether Google exists; a
  second, client-side switch drifts from it. When the provider is not configured, Better Auth answers
  `PROVIDER_NOT_FOUND` (404) on `/sign-in/social` and `/link-social`; map that code to "Google
  sign-in isn't available right now. Use your email" instead of hiding the button.
- A pure `?error=` code → copy map: `access_denied` (cancelled), `unable_to_link_account`,
  `account_not_linked`, `account_already_linked_to_different_user`, `email_does_not_match` and the
  legacy `email_doesn't_match`, `state_invalid` / `state_security_mismatch` (show as "expired, try
  again"), and a generic fallback. Never render the raw code.
- Account page, "Sign-in methods": Connect (`linkSocial`), Disconnect (`unlinkAccount({ accountId })`
  with Better Auth's row id from `listAccounts()`; disabled when it is the only method; map
  `FAILED_TO_UNLINK_LAST_ACCOUNT` and 403 `SESSION_NOT_FRESH` → "sign in again to disconnect"), and
  "Set a password" for OAuth-only users if a password client (desktop, mobile) still needs one
  (`auth.api.setPassword` is server-only; expose it through your own endpoint, 409 on
  `PASSWORD_ALREADY_SET`).
- Sign-up page: a short legal notice under the button ("By continuing with Google you accept the
  Terms and the Privacy Policy"), no checkbox.

### 3. Google Cloud Console (owner, once)

Console naming as of 2026: "Google Auth Platform".

1. A project per product, owned by the account that should own its Google assets.
2. **Branding:** app name, support email, audience **External**, developer contact. Home page,
   privacy URL, terms URL, authorized domain. **Leave the logo empty** — a logo triggers brand
   verification.
3. **Audience:** keep *Testing* and add every tester under *Test users* (at most 100). Anyone else
   is refused until you publish *In production*, which a public launch requires.
4. **Data access:** only `openid`, `userinfo.email`, `userinfo.profile`.
5. **Two Web application clients**, `<app>-web-dev` and `<app>-web-prod`, so a dev secret living in
   laptop env files cannot touch production:
   - dev: origin `http://localhost:<web-port>`, redirect `http://localhost:<web-port>/api/auth/callback/google`;
   - prod: origin `https://<web-origin>`, redirect `https://<web-origin>/api/auth/callback/google`.
6. Copy each secret straight into its store; the console may show it once.

Google accepts no wildcard redirect URIs, so Google sign-in does not work on preview deployments
(`*.workers.dev`). Register the apex only and redirect `www` to it at the edge: the state cookie is
host-only, so a flow started on `www` and called back on the apex fails.

### 4. Legal pages

The privacy policy needs a "Google sign-in and Google user data" section that matches the code:
scopes requested, data received (Google account id, email, email-verified flag, name, picture URL),
what it is used for, that no other Google API is called, no ads or model training, where it is
stored, retention and deletion, how to revoke access from the Google Account, and the Limited Use
sentence ("…will adhere to the Google API Services User Data Policy, including the Limited Use
requirements"). The terms say accounts can be created with Google and that the user is responsible
for that account's security. Bump the policy version date.

**Claim only what the code does.** If the legal copy is drafted before `encryptOAuthTokens` is on,
say "kept server-side", and revisit the wording when the linking PR lands.

### 5. Env and secrets checklist

| Value | Kind | Dev store | Production store |
| --- | --- | --- | --- |
| `BETTER_AUTH_URL` | config — the **web** origin | secret store, public/config folder, tagged for the API | deploy environment **variable** |
| `GOOGLE_CLIENT_ID` | config | same | deploy environment **variable** |
| `GOOGLE_CLIENT_SECRET` | secret | secret store, auth folder, tagged for the API | deploy environment **secret** |

- Follow [infisical-secrets.md](../infra/infisical-secrets.md): the secret in the auth folder, the
  consumer tag on every value, separate dev and prod clients.
- The deploy workflow writes the three into the Worker's secrets file and **drops empty values**, so
  merging the workflow before the values exist is safe
  ([worker-secrets-with-deploy.md](../monorepos/worker-secrets-with-deploy.md)).
- Check presence without printing: generate the consumer's env file and `grep -c GOOGLE_` it (expect
  `2`), then delete the file. Names only from `wrangler secret list`.
- Setting `BETTER_AUTH_URL` changes the base URL for every caller, including a desktop app that calls
  the API directly. Smoke-test email sign-in on web and desktop before the first release that sets it.
- No secret value in chat, tickets, PR bodies or test fixtures; tests use placeholder ids and a fake
  token endpoint.

## Tests that must exist

**The in-process OAuth harness.** Build a real `betterAuth` with `memoryAdapter`, your real provider
builder, `account` options and hooks; stub global `fetch` so the only allowed call is Google's token
endpoint (`https://oauth2.googleapis.com/token`), returning a fake `id_token` whose payload you set
per test (`{ sub, email, email_verified, name }`, any signature). Drive `/sign-in/social` → read
`state` from the returned URL → `GET /callback/google?code=fake&state=…` with a small cookie jar.
This pins Better Auth's internal ordering; an upgrade that changes it fails here.

| Case | Protects against |
| --- | --- |
| New email → one verified user, one Google account | Broken sign-up |
| Stored tokens are encrypted, not as received | Plain tokens at rest |
| Verified password user + Google → linked, password and sessions keep working | Over-eager guard |
| Unverified password user + Google → password gone, old sessions gone, new session works, notice written | Pre-hijack (option C) |
| Cleanup fails half-way (inject a failing `account.delete.before`) → sign-in refused, nothing linked | Non-fail-closed guard |
| Google `email_verified: false` + existing user → refused, no row | Unverified join |
| Cancel at Google → `access_denied` on the error URL | Raw errors |
| id-token sign-in path closed | Side door into implicit linking |
| Manual link with a different email keeps the password; same verified email marks verified | Wrong flow detection |
| Google identity already on another user → refused | Account merge |
| Unlinking the last method refused; with a password left, allowed | Lock-out |
| OAuth-only user sets a password, signs in with it; a second set is refused | Desktop lock-out |

Plus unit tests for the decision function (exhaustive), the claim parser, the error → copy map in
every locale, the button (including `PROVIDER_NOT_FOUND`) and the account controls.

## Pitfalls hit

- `BETTER_AUTH_URL` pointed at the API origin in the old docs → state mismatch on every callback.
- `verification` unmapped → OAuth state and password reset both failing.
- Destructive cleanup first planned in `create.after` → would have left the attacker's password after
  a failed cleanup.
- A path allowlist in the hook would have skipped the id-token path.
- Safari may drop `sameSite=none; secure` cookies on `http://localhost`: test locally in Chrome.
- A client feature flag for the button was added, then removed: two switches for one fact.
- Legal copy drafted while token encryption was off, then the linking PR turned it on.

## How to test it yourself

Your Google account must be a test user while the app is in Testing.

1. Start the stack with the dev Google client configured.
2. Signed out, sign-up page → Continue with Google → a throwaway account: you land on the dashboard,
   signed in, no verification email.
3. Sign up with email+password on an address you own and verify it; sign out; Google with it → same
   account, both methods work.
4. (Option C) Sign up with email+password on another address, **do not verify**, sign in on the
   desktop too. In a private window, Google with that address → the old password fails, the desktop
   is signed out on its next call, the dashboard shows the one-time notice once.
5. Account page: disconnect Google (works with a password left; disabled for an OAuth-only user);
   connect a different Google account.
6. Cancel on Google's screen → back on the sign-in page with "cancelled" copy.
7. With the API's Google values removed → the button shows "isn't available right now".

## Reference implementation

Kaipu Record, owner's product:
[#323](https://github.com/niway-dev/kaipu-record/pull/323) (env-gated provider, explicit
`BETTER_AUTH_URL`, workflow, console runbook),
[#328](https://github.com/niway-dev/kaipu-record/pull/328) (option C guard, harness, button, account
controls),
[#329](https://github.com/niway-dev/kaipu-record/pull/329) (privacy and terms for Google),
[#330](https://github.com/niway-dev/kaipu-record/pull/330) (the guard adapted to 1.7),
[#337](https://github.com/niway-dev/kaipu-record/pull/337) (feature flag removed, `PROVIDER_NOT_FOUND` copy).

## Related

- [better-auth-1-7-upgrade.md](./better-auth-1-7-upgrade.md) — version prerequisites and the 1.7 linking defaults
- [web-sign-in-flows.md](./web-sign-in-flows.md) — where the callback lands, neutral error copy
- [desktop-browser-sign-in.md](./desktop-browser-sign-in.md) — the same button on the desktop sign-in page
- [legal/privacy-policy-template.md](../legal/privacy-policy-template.md) — base text for step 4
- [infra/infisical-secrets.md](../infra/infisical-secrets.md) — where each value lives
