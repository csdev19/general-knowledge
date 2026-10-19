# Playbook: upgrade Better Auth from 1.4 to 1.7

**An upgrade PR changes the version and nothing users can see.** Pin what the apps depend on in a
contract test first, run it on the old version, then bump and fix until the same tests pass. Every
new feature that motivated the upgrade (the Electron plugin, a new provider) lands in a later PR,
with this one as its regression net.

Last reviewed: 2026-10-19, against `better-auth` 1.4.18 → 1.7.11 and `@better-auth/electron` 1.7.11.

## When to use

- You need `@better-auth/electron` (it requires `better-auth ^1.7.11`; the plugin exists from 1.5).
- You ship or are about to ship OAuth sign-in on 1.4.x. The range 1.4.18 → 1.7.11 closes, among
  others, [GHSA-965c-763c-88jm](https://github.com/better-auth/better-auth/security/advisories/GHSA-965c-763c-88jm)
  (critical, fixed in 1.7.7: an OAuth state usable like a magic link to sign in as someone else) and
  [GHSA-g38m-r43w-p2q7](https://github.com/better-auth/better-auth/security/advisories/GHSA-g38m-r43w-p2q7)
  (high, fixed in 1.6.11: OAuth sign-in linking to an account an attacker pre-registered). Read the
  advisory list for the plugins you actually use; several in this range affect only magic link,
  SSO, SCIM, Stripe, device authorization or the OAuth provider plugin.
- You want `deviceAuthorization` (RFC 8628): use ≥ 1.6.11, earlier 1.6.x let any session approve any
  pending code.

## Prerequisites and order

1. A test runner in the server app that can import `better-auth` directly.
2. An inventory of the Better Auth surface the apps use (step 1).
3. No feature work in the same branch. If a feature PR is in flight on the old version, merge it into
   the upgrade branch so its tests run on 1.7 — then the upgrade PR's diff shrinks once that feature
   lands on `main`.

## Steps

### 1. Inventory what the apps depend on

Search the server, web, desktop and any admin console for:

| Search for | Why |
| --- | --- |
| `auth.api.`, `authClient.` calls | Endpoint names, argument shapes, return shapes |
| `internalAdapter.` | Internal APIs changed signatures (see the table below) |
| `databaseHooks`, `hooks:` | Hook ordering and how a throw surfaces changed |
| Error code strings (`"USER_ALREADY_EXISTS"`, `"YOU_CANT_…"`, `?error=` maps) | Codes were renamed; the UI maps them to copy |
| `drizzleAdapter(…, { schema })` | The adapter now validates this map at runtime |
| `trustedOrigins`, custom `Origin` headers in native clients | The origin check got stricter |
| `bearer(`, `set-auth-token`, `customSession(` | Desktop and mobile clients read these directly |
| `unlinkAccount`, `setPassword`, `deleteSessions`, `deleteVerificationValue` | Known breaking changes |

### 2. Write the contract test on the current version

Create `apps/server/src/lib/auth-contract.test.ts`. Build a real `betterAuth({...})` in process with
`memoryAdapter` (from `better-auth/adapters/memory`) and drive it through `auth.handler(new Request(...))`.
No database, no network. Pin, at least:

- the production adapter config passes Better Auth's own schema check (call the adapter factory the
  way Better Auth does and assert no `SCHEMA_MISMATCH` / "model not found");
- custom password `hash`/`verify` still verify a hash stored in the old format (use a fixed stored
  hash, not a fresh round-trip only);
- sign-up and sign-in with the native client's `Origin` return the signed token in `set-auth-token`;
- `get-session` with `Authorization: Bearer <signed token>` returns the `customSession` shape;
- `bearer({ requireSignature: true })` refuses an unsigned session token;
- bearer sign-out with no cookie and no `Origin` ends the session;
- every error code a client maps (for example `PASSWORD_TOO_SHORT`) is unchanged.

Run it on the **old** version and commit it green. That commit is the definition of "no behaviour
change".

### 3. Bump in lockstep

- Move `better-auth` in the root catalog. Add every `@better-auth/*` package you use directly at the
  **same** version; they are released together.
- Drizzle users: the adapter moved to `@better-auth/drizzle-adapter`, which needs
  `drizzle-orm ^0.45.2`. Bump every workspace that declares `drizzle-orm`.
- Check the lockfile resolves **one** copy of `@better-auth/core`. Two copies (seen when the root
  pinned an older `jose` than the plugin wanted) make plugin types stop fitting `BetterAuthPlugin`.
  Fix by bumping the duplicated transitive (`jose` there) in the root, not by casting.

### 4. Fix the breaking changes

Read each against the inventory. The ones met in the reference upgrade:

| Change (version) | Symptom | Fix |
| --- | --- | --- |
| The adapter checks its `schema` map against every model Better Auth writes, before every endpoint (1.7) | **Every** auth request fails with `SCHEMA_MISMATCH`, not only the flow that uses the missing model | Map every model, `verification` included. On 1.4 a missing `verification` silently broke only password reset and OAuth state — check whether reset has been broken in production |
| Origin check: a request carrying `Origin` is always validated; a host-bearing pattern (`<scheme>://app`) must match the host exactly (1.7) | A native client sending `Origin: <scheme>:/` gets 403 | Trust the bare scheme pattern `<scheme>:/`; keep any literal older builds still send |
| `accountLinking.requireLocalEmailVerified` defaults to `true` (1.7) | Implicit OAuth link to an unverified local user answers `account_not_linked` before your hooks run | Leave the default unless you have the [pre-hijack guard](./google-sign-in.md#the-pre-hijack-guard); then set `false` deliberately, with a comment |
| `handleOAuthUserInfo` rethrows an `APIError` (1.7) | A hook that aborts a link with `APIError` (no `code`) escapes the callback as a bare 403 | Abort from hooks with a plain `Error`; Better Auth then redirects with `?error=unable_to_link_account` |
| `internalAdapter.deleteSessions` takes session **tokens** (1.7) | Passing a user id throws "Value must be an array" | `internalAdapter.deleteUserSessions(userId)` |
| `unlinkAccount` takes `{ accountId }` — Better Auth's own account row id — not `{ providerId }` (1.7) | Disconnect does nothing / 400 | Read the row id from `listAccounts()` and send it |
| Last-account unlink code is `FAILED_TO_UNLINK_LAST_ACCOUNT` (was `YOU_CANT_UNLINK_YOUR_LAST_ACCOUNT`) | Raw code shown | Remap |
| `setPassword` on an account that has one answers `code: "PASSWORD_ALREADY_SET"` (the 1.4 message is gone) | Your 409 mapping stops matching | Match the code |
| `deleteVerificationValue(id)` removed | Type error | `consumeVerificationValue(identifier)` reads and deletes in one race-safe step |
| OAuth error `email_doesn't_match` renamed `email_does_not_match`; new `state_invalid`, `state_security_mismatch` | Raw codes on the error page | Map the new names, keep the old one for links in flight |
| A disabled id-token path (`disableIdTokenSignIn`) answers 404 instead of 401 | Test expectation only | Update the test; the path is still closed |
| `freshSessionMiddleware` measures age from `createdAt` (1.6/1.7), default `freshAge` 1 day | Unlink and `list-sessions` answer 403 `SESSION_NOT_FRESH` for sessions signed in more than a day ago | Show "sign in again to do this" copy |
| `sensitiveSessionMiddleware` reads the session from the database (1.7) | — | Nothing, unless you relied on the cookie cache there |
| `create.after` hooks are queued until after a transaction (1.5) | Only inside `runWithTransaction` (a brand-new OAuth user) | Do not do anything in `after` that the next request depends on |
| Duplicate `(providerId, accountId)` account rows are rejected (1.7) | — | Add a unique index on `account (provider_id, account_id)` so the database agrees |
| Drizzle: an invalid affected-row count throws (1.7) | Writes fail on drivers that do not report row counts | `neon-http` reports them (`fullResults`); check other drivers |

Unchanged in the reference upgrade (verified from source and tests): the email sign-in/up,
`get-session` and `sign-out` paths, the `set-auth-token` header, `customSession` shape,
`getSession({ query: { disableCookieCache: true } })`, the `cookieCache` default strategy, and
`getOAuthState()` exported from `better-auth/api`.

### 5. Schema

Run Better Auth's schema diff from the contract test against `user`, `session`, `account`,
`verification`. Going straight from 1.4.18 to 1.7.11 needs no new column. Apply the unique index with
the repository's usual schema command (`db:push` or a migration).

### 6. Verify

Run the repository's full gate without the build cache, and read the test counts from that run. Then
do the manual pass below on a dev database.

## Tests that must exist

| Test | Failure it protects against |
| --- | --- |
| Adapter schema check | All auth requests failing after deploy |
| Fixed stored password hash verifies | Every existing password user locked out |
| Native `Origin` accepted, look-alike origins (`https://evil.example`, `<scheme>://app.evil.example`, `<scheme>://other`) refused | A desktop that cannot sign in, or a too-loose pattern |
| Signed bearer accepted, unsigned refused | Session-id forgery through the bearer plugin |
| Each feature's own flow tests (OAuth linking, desktop exchange) running on the new version | Hook ordering changes nobody noticed |

## Pitfalls hit

- Mapping `verification` fixed a production bug nobody had reported: password reset had been failing
  on 1.4 for lack of it.
- The temptation to "also clean up" a deprecated option in the same PR. Do not: the upgrade is
  reviewable only because its diff is version + fixes.
- Two copies of `@better-auth/core` produce type errors that look like a plugin bug.

## How to test it yourself

1. Check out the branch, install, push the schema to a **dev** database, start the stack.
2. Web: sign up, sign out, sign in. Change the password from the profile page; sign in with the new one.
3. Forgot password → open the emailed link → set a new password → sign in.
4. Desktop (if any): sign in, quit, relaunch — the stored session is restored.
5. OAuth (if shipped): sign in with the provider, cancel at the provider (you return with copy, not a
   raw code), disconnect from the profile page (a session older than a day asks you to sign in again).

## Reference implementation

Kaipu Record, owner's product: [PR #330](https://github.com/niway-dev/kaipu-record/pull/330)
(upgrade, contract test `auth-contract.test.ts`, the table above with source line references) and
[PR #333](https://github.com/niway-dev/kaipu-record/pull/333) (the `jose` duplicate and the
bare-scheme origin, found while adding the Electron plugin).

## Related

- [google-sign-in.md](./google-sign-in.md) — the linking options the 1.7 defaults interact with
- [desktop-browser-sign-in.md](./desktop-browser-sign-in.md) — the feature that requires 1.7.11
- [README.md](./README.md) — order of the auth playbooks
