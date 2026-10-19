# Auth playbooks

Step-by-step recipes for account sign-in on **Better Auth** across a server, a web app and an
Electron desktop app: upgrading Better Auth, adding Google, signing the desktop in through the
system browser, and the email flows around a password. Each playbook is written to be replayed in
another product: placeholders in the rules, the original product's values only in clearly marked
examples, and the reference implementation linked at the end.

The topology they assume is the hub default — a web Worker that proxies `/api/auth/*` to an API
Worker, so the browser only ever talks to the web origin ([api/hono.md](../api/hono.md)). When
identity spans several products in separate repositories, read
[centralized-auth-service.md](../api/centralized-auth-service.md) first; the cookie and origin rules
there change several steps below.

## Playbooks

| Playbook | Use it when |
| --- | --- |
| [better-auth-1-7-upgrade.md](./better-auth-1-7-upgrade.md) | You are on Better Auth 1.4–1.6 and need 1.7 (the Electron plugin, or the OAuth security fixes). The breaking changes that bit, and the contract test that makes the upgrade a no-behaviour-change PR. |
| [google-sign-in.md](./google-sign-in.md) | You add "Continue with Google". Account-linking policy and the pre-hijack guard, Google Cloud Console, consent screen and legal pages, env and secrets, the in-process OAuth test harness. |
| [desktop-browser-sign-in.md](./desktop-browser-sign-in.md) | An Electron app must sign in without a password field: the system browser, a custom-scheme deep link, a single-use code bound to PKCE and state, the paste-code fallback, one single-instance lock. |
| [web-sign-in-flows.md](./web-sign-in-flows.md) | You build the web sign-in pages around email: verification by link, resend, forgot/reset password, enumeration-safe errors, the desktop sign-up return, and where every sign-in lands (`/dashboard`). |

## Order

```
better-auth-1-7-upgrade ──► google-sign-in ──► desktop-browser-sign-in
          │                       │                      ▲
          └──────────► web-sign-in-flows ────────────────┘
```

- The upgrade comes first and alone. The Electron plugin needs Better Auth 1.7.11, and the Google
  work is safer on 1.7 (OAuth-state and pre-linking advisories were fixed in 1.6.11 and 1.7.7).
- Google before the desktop flow: the desktop sign-in page reuses the web's Google button and its
  error mapping, so building Google first means the desktop gets it for free.
- The web flows (post-sign-in landing, neutral errors) can go in parallel with Google; the desktop
  pages are built on the same forms.

## Process lessons from the reference implementation

These cost time once; they are not specific to auth, but every playbook here was shipped as a stack
of pull requests, so they apply.

- **A stacked PR merged into its base branch is not on `main`.** Two PRs of the reference stack were
  merged into the branch below them after that branch had already landed, so their code sat on a
  dead branch and needed a carrier PR. Retarget each PR to `main` as soon as the one below it
  merges, before merging it, and check `git log origin/main` after the last merge rather than
  trusting the "Merged" badge.
- **Count tests without the build cache.** A cached `turbo run test` replays the last result; when a
  PR body quotes test counts, take them from an uncached run (`--force`, or `CI=true` with the cache
  off) so the number belongs to this commit.
- **Never leave the suite red.** A flaky unrelated test that blocks a PR is fixed in its own small PR
  right away (the reference stack paused for one: an assertion made in the same tick as a passive
  `useEffect`), not skipped.
- **Secrets never travel in chat, tickets or PR bodies.** Each playbook names where a value lives
  and how to check it exists without printing it.

## Related

- [api/centralized-auth-service.md](../api/centralized-auth-service.md) — the cross-origin topology (cookies, `trustedOrigins`, CORS) when identity spans products
- [api/gotchas/better-auth-cross-origin.md](../api/gotchas/better-auth-cross-origin.md) — immutable request headers on Workers
- [convex/better-auth.md](../convex/better-auth.md) — Better Auth hosted inside Convex instead of a Worker
- [infra/infisical-secrets.md](../infra/infisical-secrets.md) — folders, consumer tags and the CI boundary the env checklists follow
- [infra/transactional-email.md](../infra/transactional-email.md) and [packages/transactional-email-playbook.md](../packages/transactional-email-playbook.md) — the sender behind verification and reset emails
- [legal/README.md](../legal/README.md) — the privacy and terms templates the Google consent screen needs
- [desktop/main-process-architecture.md](../desktop/main-process-architecture.md) and [desktop/ipc-contract.md](../desktop/ipc-contract.md) — where the desktop half plugs in
