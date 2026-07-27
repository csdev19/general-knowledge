# Custom Domain Migration (registrar → Cloudflare Workers)

_Pointing a real domain at a Worker for the first time: the DNS records that block it, the deploy failure they cause, and everything that breaks **after** the domain works._

The domain part is a 5-minute job. The reason it costs an afternoon is that a
domain change quietly invalidates every place the app's origin is hardcoded —
auth, OAuth, CORS, social cards, deep links — and each of those fails
*separately*, after the previous one is fixed.

---

## The end state

```jsonc
// apps/web/wrangler.jsonc
{
  "name": "app-web",
  "routes": [
    { "pattern": "example.app", "custom_domain": true },
    { "pattern": "www.example.app", "custom_domain": true },
  ],
}
```

`custom_domain: true` means **Wrangler creates and owns the DNS record** on every
deploy. No manual CNAME. That is the convenience — and the source of the
failure below, because Wrangler refuses to silently overwrite a record it did
not create.

---

## Two independent things, often confused

Moving a domain "to Cloudflare" is two separate migrations. Doing one and
assuming the other happened is the usual first mistake.

| | What it is | How to verify |
| --- | --- | --- |
| **Nameservers** | The registrar delegates DNS to Cloudflare. Done once, at the registrar. | `dig +short NS example.app` → `*.ns.cloudflare.com` |
| **DNS records** | The actual A/CNAME/MX rows inside the Cloudflare zone. Cloudflare **imports** whatever the old provider had. | Cloudflare dashboard → DNS |

The registrar transfer (who bills you for the domain) is a **third**, unrelated
thing. It does not need to happen for any of this to work.

---

## The trap: imported parking records

When the zone is created, Cloudflare scans the old provider's DNS and copies it.
If the domain was never used, what it copies is the registrar's **parking page**:

| Name | Type | Content | Proxy |
| --- | --- | --- | --- |
| `example.app` | A | `192.64.119.28` | Proxied |
| `www.example.app` | CNAME | `parkingpage.<registrar>.com` | Proxied |

These are live, proxied records on exactly the two hostnames the Worker wants to
claim. Two things follow.

### Symptom 1 — the site returns 522 / 525, not 404

```
$ curl -sI https://example.app | head -1
HTTP/2 522
```

| Code | Meaning | Reads as |
| --- | --- | --- |
| **522** | Cloudflare could not open a TCP connection to the origin | The A record points somewhere dead |
| **525** | TLS handshake with the origin failed | The origin exists but has no cert for this host |

A Worker custom domain **can never return 522/525** — there is no origin to
connect to; the Worker *is* the origin. So a 5xx in the 52x family is proof the
hostname is still pointing at a legacy record, not at the Worker.

### Symptom 2 — the deploy fails on the trigger step

```
Uploaded app-web (4.57 sec)              ← the Worker itself deployed fine
No targets deployed for app-web          ← no custom domain was attached
✘ [ERROR] Some triggers failed to deploy for app-web:
    - A request to the Cloudflare API
      (/accounts/***/workers/scripts/app-web/domains/records) failed.
```

Read those three lines in order — they say the code shipped and only the
*routing* failed. Locally, Wrangler prompts `Would you like to override existing
DNS records?`. **In CI there is no TTY**, so the prompt cannot be answered, the
API call is rejected, and the error surfaces with no code and no body.

> The empty error message is the tell. A real permissions or quota failure comes
> back with a Cloudflare error code (`[code: 10000]`). An error with *no*
> detail is almost always the non-interactive override prompt.

---

## Diagnosis, in order

```bash
# 1. Is the zone even on Cloudflare?
dig +short NS example.app

# 2. What is each hostname actually pointing at?
dig +noall +answer example.app A
dig +noall +answer www.example.app A www.example.app CNAME

# 3. What does the edge return?
curl -s -o /dev/null -w "%{http_code}\n" https://example.app
```

`NS` on Cloudflare + proxied A records + a 52x response = imported legacy
records. That is the whole diagnosis.

---

## The fix

1. **Delete** the two conflicting rows in the Cloudflare DNS dashboard: the apex
   `A` and the `www` `CNAME`.
2. **Re-run the deploy.** Wrangler recreates both hostnames as Worker custom
   domains and they reappear in the DNS table as type `Worker`.

Equivalent manual path: Workers → *script* → Settings → Domains & Routes → Add
custom domain. The dashboard *does* offer the override prompt, and once the
domains exist, subsequent CI deploys are idempotent no-ops.

### Do NOT delete the whole zone's records

The parking-page import usually arrives alongside records that are **still
live**:

| Keep | Why |
| --- | --- |
| `MX` (e.g. `eforward*.registrar-servers.com`) | Email forwarding for `@example.app`. Deleting these silently drops all inbound mail. |
| `TXT` `v=spf1 …` | SPF. Deleting it makes your outbound mail fail authentication. |
| `TXT` `_dmarc`, `*._domainkey` | DMARC / DKIM, same. |

Delete **only** the two records on the hostnames the Worker is claiming.

---

## Surfacing the real error in CI

Because the failure message is empty by design, add this to the deploy step
before debugging blind:

```yaml
- name: Deploy
  uses: cloudflare/wrangler-action@v3
  env:
    WRANGLER_LOG: debug   # prints the raw Cloudflare API response body
```

### API token scope

If the domains still fail after the conflicting records are gone, it is the
token. Attaching a custom domain touches the **zone**, not just the account:

| Permission | Needed for |
| --- | --- |
| `Account → Workers Scripts → Edit` | Uploading the Worker (this one is usually already there — it is why the upload succeeds) |
| `Zone → Zone → Read` | Resolving the hostname to a zone |
| `Zone → DNS → Edit` | Creating the record |

A token with only the first permission produces exactly this failure mode: a
green upload and a failed trigger.

---

## apex vs `www` — decide, do not serve both

Serving the app on both hostnames is the tempting default and it is a trap,
because auth is **origin-scoped**. A session started on `www.example.app` is not
sent to `example.app`: same app, two disconnected login states.

Pick one:

- **Canonical apex, `www` redirects** (recommended). One origin, one session, and
  it matches the `<link rel="canonical">` you should already be emitting. Do the
  301 in the Worker or as a Cloudflare Redirect Rule.
- **Both serve the app.** Then every origin allowlist in the stack must contain
  *both* hostnames, forever, and you accept the split-session behaviour.

---

## Post-migration checklist — what a new origin breaks

This is the part that costs the time. Each item fails independently and only
once you actually exercise it.

| # | Thing | Where it lives | Failure if missed |
| --- | --- | --- | --- |
| 1 | **Auth base URL / trusted origins** | Backend env var (`SITE_URL`) → `baseURL` + `trustedOrigins` | `Invalid origin` on every login |
| 2 | **OAuth redirect URIs** | Google / GitHub / Apple consoles | `redirect_uri_mismatch` |
| 3 | **CORS allowlist** | API worker config | Preflight failures on every mutation |
| 4 | **Canonical + social absolute URLs** | `<link rel="canonical">`, `og:url`, `og:image` | Broken link previews, duplicate-content SEO |
| 5 | **PWA manifest** | `start_url`, `scope`, `id` | Installed app opens the old origin |
| 6 | **Mobile deep links** | `WEB_BASE_URL`-style env var; Universal Links / App Links association files | Invite/share links open the browser instead of the app |
| 7 | **Transactional email links** | Verification / reset / invite templates | Users land on a dead host |
| 8 | **Third-party key restrictions** | Maps/analytics keys restricted by HTTP referrer | Map tiles or analytics silently stop |

**Item 1 is the one that always gets missed**, because it is usually the only
one that lives *neither in the repo nor in CI* — it is an env var set in a
backend dashboard, so a repo-wide grep for the old hostname does not find it.

```ts
// The single source of the trusted origin — set in the backend's dashboard,
// invisible to `grep`.
const siteUrl = process.env.SITE_URL!;   // https://example.app  (no trailing slash)

betterAuth({
  baseURL: siteUrl,
  trustedOrigins: [siteUrl /* , "https://www.example.app" if both serve */],
});
```

Origins are compared **exactly**: no trailing slash, and `https://example.app`
≠ `https://www.example.app`.

---

## Order of operations

1. Nameservers → Cloudflare; confirm with `dig NS`.
2. Audit the imported records. Delete the parking rows, keep mail.
3. Add `custom_domain` routes to the Worker config; deploy.
4. Confirm both hostnames return the app (not 52x).
5. Decide apex vs `www`; add the redirect.
6. Walk the post-migration checklist **before** announcing the new URL.

Steps 1–4 are the domain. Step 6 is the actual migration.
