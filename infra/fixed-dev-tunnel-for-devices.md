# A fixed development tunnel: the phone reaches the local API at one URL

A mobile app on a physical device cannot reach `localhost`. The usual answers each put a manual
step in front of "edit, reload, see it": a LAN IP that changes with the network, an ad-hoc tunnel
whose URL changes on every start, or a deploy per change. The last one is the expensive loop:
with AI-assisted development there are many small changes per hour, and deploying each one is
exactly the budget to protect.

The pattern here gives the device **one HTTPS URL that never changes**, pointed at whatever the
laptop is running, started and stopped by the same command that starts the dev servers. It is the
local-first counterpart of Convex's `dev` deployment: Convex gets a stable URL by deploying on every
save, which works because its deploy takes a second. A Worker deploy does not, so the stable URL
has to come from a tunnel instead.

## The rules, and the fact behind each one

1. **A fixed URL needs a zone you control.** Cloudflare's quick tunnels (`*.trycloudflare.com`) are
   the only domain-less option and they get a new random hostname on every start, so a value stored
   in a secrets manager is stale by the next session. A named tunnel binds a stable hostname, but
   only on a zone in your account. ngrok's free static domain is the third-party equivalent (one
   fixed random hostname), with a bandwidth cap and a browser interstitial to test against your
   client's `fetch`.

2. **The hostname is a random first-level subdomain: `<app>-<hash>.<zone>`, never `<hash>.<app>.<zone>`.**
   The free Universal SSL certificate covers `<zone>` and `*.<zone>`, one level. A second level has no
   certificate and fails the TLS handshake; covering it costs Advanced Certificate Manager. Putting the
   app in the prefix keeps the pattern repeatable across apps for free. A first-level name under a
   wildcard also does not appear as its own entry in Certificate Transparency logs, which is how
   subdomains are usually discovered. Ten hex characters are enough; the point is "not guessable",
   not "secret".

3. **Tunnel and zone live in the same Cloudflare account.** A `<uuid>.cfargotunnel.com` target only
   proxies DNS records from the account that owns the tunnel. If the production Workers live in a
   different account than the zone, the tunnel follows the zone.

4. **Let the dev server own the tunnel.** `wrangler dev --tunnel --tunnel-name <name>` (Wrangler ≥ 4.1xx)
   starts the tunnel after the local server is up, downloads `cloudflared` itself, and stops it with
   the server. No launch agent, no `config.yml` outside the repo, nothing to replicate on a second
   machine. The alternative, a standalone `cloudflared` service, is always up, which also means it
   serves public 502s all day while nothing listens.

5. **Wrangler needs a remotely managed tunnel with an exact ingress match.** It lists tunnels by
   name, reads the tunnel configuration stored on Cloudflare, and keeps the ingress rules whose
   `service` equals its own origin (`http://localhost:<port>`, after normalising `127.0.0.1` and
   `0.0.0.0`). No matching rule, no tunnel. Create the tunnel with `config_src: "cloudflare"` and an
   ingress of `[{hostname, service: "http://localhost:<port>"}, {service: "http_status:404"}]`.

6. **Tunnel operations take an API token, not `wrangler login`.** Wrangler rejects the OAuth
   session for every tunnel call ("Cloudflare Tunnel commands require API token authentication").
   Give it `CLOUDFLARE_API_TOKEN` and `CLOUDFLARE_ACCOUNT_ID` through the app's one `.env`, which
   Wrangler reads for itself. The same file also feeds the local Worker's `env`, so the dev-scoped
   token is visible to local Worker code; production never receives it. Do not split it into a
   `.dev.vars` to hide it: Wrangler then reads `.dev.vars` **instead of** `.env`, and two files
   become two sources of truth where the hidden one silently wins. One token with
   *Account → Cloudflare Tunnel → Edit* (Wrangler fetches the tunnel token) and *Zone → DNS → Edit* on
   the one zone (for rotation) serves both the dev server and the rotate command. This also removes
   the question of which account each laptop is logged into.

7. **The hostname lives in the secrets manager, not in the repo.** It is the mobile app's API URL
   (`EXPO_PUBLIC_API_URL` in the `dev` environment), written into the app's `.env` by the same pull
   that writes every other local value. The repo holds only the non-secret coordinates: account id,
   zone, tunnel name, hostname prefix, local origin. Pulling values is the one moment local
   development talks to the secrets manager; `dev` itself never does.

8. **Rotate on demand, by one command, never on a calendar.** Scheduled weekly, rotation is
   recurring toil for a server that is down most of the day and already requires a session. Make it
   one idempotent command (`tunnel:rotate`) so it costs a minute when wanted; a cron can call it
   later if the posture changes. Order the steps so a failure anywhere leaves a working URL in the
   app: check the secrets-manager key is reachable **before** touching anything; serve old **and**
   new hostnames from the ingress; create the new CNAME; update the secrets manager; only then
   delete the old `<app>-*` records and narrow the ingress to the new hostname; re-pull; restart
   the bundler. Replacing the ingress first, as a first draft did, makes the old URL answer 404
   the moment the next step fails.

9. **One tunnel, one connector at a time.** Two laptops running the same named tunnel at once
   load-balance requests between them with no error. Either keep the rule "stop here before starting
   there", or give each person a tunnel (`<app>-dev-api-<person>`) and a hostname.

10. **Name the exposure.** While the dev server runs, anyone who knows the hostname reaches the
    local API against the dev database. What makes that acceptable, and what would end it (external
    testers, real data), belongs in the project's ADR, with Cloudflare Access plus a service token in
    the dev build as the mitigation when it is no longer acceptable.

## What to check on the first run

- Better Auth (or any auth library that infers its base URL from the request) sees the public
  hostname and `X-Forwarded-Proto: https` through the tunnel. If it infers `http://localhost` instead,
  pin the base URL to the tunnel URL in the `dev` environment.
- Nothing in CORS has to change for a native client that sends `exp://` / `<scheme>://` origins; the
  tunnel only changes the hostname the client dials.
- Metro inlines `EXPO_PUBLIC_*` at bundle time: after a rotation, restart the bundler.

## Measured example

GymTrack (owner's product), 2026-10-26. Zone `niway.dev`, hostname `gymtrack-<10 hex>.niway.dev`,
tunnel `gymtrack-dev-api`, origin `http://localhost:3000`. Creating the remotely managed tunnel, its
ingress and the proxied CNAME took three API calls; the hostname answered HTTP 530 / error 1033
("tunnel not connected") over a valid `*.niway.dev` certificate before any connector ran, which is the
check that DNS, certificate and routing are wired. Wrangler 4.110.0 with `wrangler login` only
printed "Cloudflare Tunnel commands require API token authentication" and the dev run ended; that
run also had a Worker startup error, so whether the tunnel failure alone stops the server is not
separated. The production Workers of the same project live in a different Cloudflare account than
the zone, which is what made rule 3 and rule 6 matter. Decision record: the project's ADR 0003.

## Related

- [`infisical-secrets.md`](./infisical-secrets.md): folders by risk, tags by consumer; the token
  above is one more consumer tag (`tunnel-tools`) in a `/cloudflare` folder.
- [`deploy-environments.md`](./deploy-environments.md): when a deployed pre-production tier is worth
  it. This page is the answer for "I need a URL, not an environment".
- [`../mobile/expo-dev-builds-and-metro.md`](../mobile/expo-dev-builds-and-metro.md): how the
  device reaches **Metro**; this page is how it reaches the **API**.
- [`../monorepos/worker-secrets-with-deploy.md`](../monorepos/worker-secrets-with-deploy.md): the
  production side of the same Worker.
