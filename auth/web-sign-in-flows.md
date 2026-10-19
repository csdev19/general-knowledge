# Playbook: web sign-in flows around email

**Email verification is a link, it never blocks, and the sign-in form never says whether an address
has an account.** Signing up with email signs the person in straight away, unverified; the
verification link only unlocks the features that need a proven inbox. Wrong password and unknown
email get one message. Every successful sign-in lands on one page, decided in one function.

Last reviewed: 2026-10-19, against `better-auth` 1.7.11.

## When to use

You offer email+password (or any email-based method) on the web pages of a Better Auth app and need
verification, resend, forgot/reset password, neutral errors, a desktop sign-up that returns to the
app, and a consistent post-sign-in destination. If your product has **no** passwords (sign-in only by
OAuth and an emailed link or code), the verification and reset sections do not apply; the
enumeration, desktop-return and landing sections still do.

## Prerequisites

- A transactional email sender ([playbook](../packages/transactional-email-playbook.md),
  [sender convention](../infra/transactional-email.md)).
- `verification` mapped in the adapter (reset and verification tokens live there).
- A public web URL in the server's env, used to build every link in an email.

## Decisions and their reasons

| Decision | Why | Rejected |
| --- | --- | --- |
| Verify by **link** | It works later, in any browser, on any device; the person does not have to keep the sign-up tab open | A 6-digit OTP for verification: forces the person to stay on the page and type, for a step that gates nothing at sign-up |
| Sign-up signs in **unverified**; verification only unlocks gated features (cloud storage, sharing) | No dead end between sign-up and first use | Blocking sign-in until verified |
| A desktop sign-up **returns to the app at once** | The code is minted at sign-up; making the person verify first strands the app waiting | Verifying before the hand-off |
| **One error** for wrong password and unknown email | No account enumeration from the sign-in form | "There's no account for this email — create one?" |
| **One post-sign-in function**, default `/dashboard` | Every entry point (forms, OAuth callback, guards) agrees | Each form choosing its own redirect |

An emailed **code** is still the better choice for *sign-in* (not verification) when the email may
be opened on another device than the one signing in — a code typed into the waiting tab completes
there, a link completes wherever it is opened. That trade-off matters most for
[desktop sign-in through the browser](./desktop-browser-sign-in.md).

## Steps

### 1. Verification (server)

```ts
emailVerification: {
  sendOnSignUp: true,
  autoSignInAfterVerification: false,   // they are already signed in
  expiresIn: 60 * 60 * 24,              // 24 h
  sendVerificationEmail: async ({ user, token }, request) => { /* build URL, send */ },
}
```

- Build the URL yourself from the public web URL (`<web-origin>/auth/email-verified?token=…` or
  your route), with the locale from `Accept-Language`.
- When email is not configured: log one error with the user id (never the link) and return.
- When the provider fails: log the cause and **rethrow**. Better Auth otherwise swallows hook errors
  and answers 200, and the person waits for an email nobody sent.
- `/auth/email-verified`: success, or "this link expired or was used" with a button that sends a new
  one. It must keep working for every link already sent, whatever the auth pages look like later.

### 2. "Check your email" after sign-up (web)

Shown in place after a successful email sign-up: "We sent a link to `<email>`. Open it to activate
<gated feature>." With:

- **Resend** with a visible 60 s countdown (the same `sendVerificationEmail`), plus a server-side
  rate limit — the countdown is UX, not protection;
- "Open Gmail" when the address is a Gmail address (optional, cheap);
- **Wrong email? Change it**: signs the just-created account out and reopens the form with the
  details filled in;
- **Go to <your library>** (the post-sign-in destination) — the page never traps the person.

The same unverified-email notice (resend + cooldown) appears on the dashboard and the profile page;
extract it once and share it.

### 3. Forgot and reset password

```ts
emailAndPassword: {
  resetPasswordTokenExpiresIn: 60 * 60,  // one hour, single use
  revokeSessionsOnPasswordReset: true,
  sendResetPassword: async ({ user, token }, request) => { /* same logging and rethrow rules */ },
}
```

- `/auth/forgot-password` answers **"Check your email" for any address**, existing or not.
- `/auth/reset-password?token=…` sets the new password and sends the person to sign in. All sessions,
  including desktop bearer sessions, end.
- Reset fails for every user if `verification` is not mapped — on Better Auth 1.4 silently. A test
  that runs reset end to end on the in-process instance is the guard.

### 4. Enumeration-safe sign-in errors

- Map `INVALID_EMAIL_OR_PASSWORD` **and** any "user not found" code to one string: "Wrong email or
  password."
- Under it, a neutral link that is true for both cases: "New to <App>? Create an account with this
  email". It opens the sign-up tab with the email prefilled.
- Carry the prefill in **module state for one navigation**, not in the URL, so the address never
  reaches the address bar, history or server logs; read it once.
- Forgot password: same answer for every address (step 3).
- Residual signal, accepted: sign-up with an existing email shows a generic "We couldn't create your
  account" and Better Auth's status still differs. Rate-limit sign-up if that matters for you.

### 5. Sign-up from the desktop returns at once

On the desktop sign-in page with `intent=sign-up`, the sign-up request carries the desktop query, so
the server mints the hand-off code at sign-up. Navigate straight to the done page with a marker
(`/auth/desktop/done?…&check_email=1`): the page opens the app as usual and adds a small "check your
email" note. The desktop's own "Resend verification email" keeps sending the same link (if it calls
the API directly, its `Origin` must stay trusted).

### 6. Where every sign-in lands

- `lib/post-sign-in.ts`: `DEFAULT_SIGNED_IN_PATH = "/dashboard"` and
  `resolvePostSignInPath(search)` → a **safe same-origin `redirect`** if present, else the default.
  Safe means: a path starting with one `/`; refuse absolute URLs, `//host`, backslashes and control
  characters.
- Used by: the sign-in and sign-up forms, the Google button's `callbackURL`, and the `/auth/login` /
  `/auth/signup` guards (a signed-in visitor opening them is bounced to the same place). Desktop flows
  skip it: they always go to the done page.
- Optional, for an SSR landing: a **server-only** `beforeLoad` on `/` that redirects to the dashboard
  when the request carries the session cookie **and** the session is valid. No cookie → render the
  landing with no API call (crawlers, new visitors). Cookie but invalid or lookup error → render the
  landing; never a redirect to login, never a loop. In-app navigation to `/` (sign-out) is not
  redirected. Match `better-auth.session_token` with an optional `__Secure-` prefix; ignore the
  cookie-cache cookie. **Skip this on a prerendered landing** — there is no server request to check.
- The header logo links to the dashboard when signed in and to `/` when signed out, so it does not
  depend on the redirect.
- Account URLs that existed before the dashboard redirect into `/dashboard/*`, so old links keep working.

### 7. Page structure

- Sign-in and sign-up are **tabs that are links**: each has its own URL and keeps every query
  parameter it came with (`redirect`, or the desktop `client_id`/`state`/`code_challenge`).
- OAuth first, "Continue with email" second; the email form reveals in place.
- `?error=<code>` from an OAuth round trip renders mapped copy (see
  [google-sign-in.md](./google-sign-in.md#2-web-appsweb)); a closed or cancelled consent is its own
  friendly message.

## Tests that must exist

| Test | Protects against |
| --- | --- |
| Wrong password and unknown email render the identical message, and no "no account" wording exists | Enumeration regressions |
| The create-with-this-email link prefills sign-up, and the email is not in the URL | Leaking addresses into logs |
| Resend countdown 1:00 → … → Resend; "change it" signs out and refills | Trapped users |
| Forgot password answers the same for any address | Enumeration via reset |
| Reset and verification run end to end on the in-process instance | The unmapped-`verification` break |
| Desktop sign-up goes to the done page with the note and still deep-links | Verification blocking the hand-off |
| `resolvePostSignInPath`: valid path kept; `https://evil.example`, `//evil`, `\\evil`, control chars → default | Open redirect |
| `/` redirect: no cookie → no lookup; stale cookie → landing; valid → dashboard | Loops and wasted API calls |
| Tabs keep `redirect` and the desktop query | Lost hand-off |
| Every string in every locale (parity test) | Untranslated auth copy |

## Pitfalls hit

- A design showed "There's no account for this email — create one" — replaced by the neutral pair.
- Better Auth swallowing a failed email send and answering 200.
- Forgot password broken in production on 1.4 for lack of the `verification` mapping.
- Calling `router.invalidate()` before navigating to the desktop done page flashed the "Continue as"
  screen; navigate directly.
- When email is not configured in dev, no verification token is created at all; the account still
  works, only the gated feature waits.

## How to test it yourself

1. Sign in with a real account's email and a wrong password → "Wrong email or password." and the
   create-account link. An address with no account → identical. The link opens sign-up with the
   email filled; the URL does not contain it.
2. Sign up with an address you can read → "Check your email" with Resend counting down and "Go to
   your library". Open the link → `/auth/email-verified` → the dashboard. The gated feature unlocks.
3. Forgot password with any address → the same "Check your email". Open the real link → set a new
   password → old sessions (web and desktop) are gone.
4. From the desktop app, "Create account" → sign up → you land on the done page, the app opens, and
   the note says to check your email.
5. Signed out, open `/dashboard` → login with `redirect=/dashboard`; sign in → back there.
   `/auth/login?redirect=https://evil.example` → the dashboard, not the external site.
6. Signed in, open `/` with a full load → the dashboard (SSR landing only). Sign out → the landing,
   no loop.

## Reference implementation

Kaipu Record, owner's product:
[#325](https://github.com/niway-dev/kaipu-record/pull/325) (dashboard hub, `resolvePostSignInPath`,
server-only `/` redirect, shared verification notice),
[#340](https://github.com/niway-dev/kaipu-record/pull/340) (auth redesign: neutral errors, check your
email by link, desktop done page, tabs as links),
[#338](https://github.com/niway-dev/kaipu-record/pull/338) (dashboard design brief).

## Related

- [google-sign-in.md](./google-sign-in.md) — OAuth error copy and the callback destination
- [desktop-browser-sign-in.md](./desktop-browser-sign-in.md) — the done page these flows hand off to
- [packages/transactional-email-playbook.md](../packages/transactional-email-playbook.md) — the sender behind every email here
- [web/server-functions.md](../web/server-functions.md) — the server function behind the `/` redirect
