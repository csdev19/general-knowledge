# CI/CD pipelines — releases & deploys

_How independent deployables in a monorepo are versioned and shipped — tag-driven, independent pipelines on GitHub Actions, deploying to Cloudflare (Workers + R2)._

A monorepo can ship **multiple independent deployables**, each versioned and released on its
own. A clean pattern is to have **no `production` branch** — everything is **tag-driven**: you
push a component-prefixed tag and the matching GitHub Actions workflow runs.

Example with three deployables (a desktop app, a web app, an API):

| Component                 | Tag pattern      | Workflow                                | Target                                                  |
| ------------------------- | ---------------- | --------------------------------------- | ------------------------------------------------------- |
| **Desktop** (Electron)    | `desktop-v*.*.*` | `.github/workflows/release-desktop.yml` | Signed macOS DMG/zip → Cloudflare R2 + a GitHub Release |
| **Web** (landing + proxy) | `web-v*.*.*`     | `.github/workflows/release-web.yml`     | Cloudflare Worker (`<app>-web`) → `<app>.example`       |
| **API** (backend)         | `api-v*.*.*`     | `.github/workflows/release-api.yml`     | Cloudflare Worker (`<app>-api`)                          |

Each also has a `workflow_dispatch` trigger, so you can run any of them manually from the **Actions**
tab without cutting a tag.

## Versioning

- The components version **independently** — the tag prefix decides what ships, and each app's
  `package.json` `version` tracks its own line.
- Start at **`0.1.0`** and move `0.1 → 0.2 → …` while pre-1.0. The first real tags are
  `desktop-v0.1.0`, `web-v0.1.0`, `api-v0.1.0`.
- Plain semver: `MAJOR.MINOR.PATCH`. The tag must match `*-v*.*.*` (three dotted numbers after the
  prefix), e.g. `desktop-v0.2.1`.

## Cutting a release

```bash
# Desktop — builds, signs, notarizes the macOS app, uploads to R2 + GitHub Release
git tag desktop-v0.1.0 && git push origin desktop-v0.1.0

# Web — builds and deploys the web Worker
git tag web-v0.1.0 && git push origin web-v0.1.0

# API — deploys the backend Worker
git tag api-v0.1.0 && git push origin api-v0.1.0
```

Keep the tag and the component's `package.json` version in sync (bump the file, commit, then tag).
Better yet, automate this with release-please so the tag and the built version can never drift —
see [release-automation.md](./release-automation.md).

## What each pipeline does

### Desktop — `release-desktop.yml`

1. Builds the macOS app for **arm64 + x64**, **signs with Developer ID** and **notarizes** (secrets in
   the `production` environment).
2. Uploads the update feed (`*-mac.zip` + `.blockmap` + `latest-mac.yml`) to R2 under `updates/`, and
   the installers (`.dmg`) to `download/<version>/` and `download/latest/`.
3. Attaches the artifacts to a **draft pre-release** GitHub Release.

The version is parsed from the tag: `desktop-v0.1.0` → `0.1.0` (used for the DMG filenames).

### Web — `release-web.yml`

1. Installs deps, builds the shared `@<scope>/*` packages, then builds the web app.
2. Deploys the Worker via `cloudflare/wrangler-action`. The Worker is attached to `<app>.example` +
   `www.<app>.example` (custom domains in the web app's `wrangler.jsonc`), so DNS + SSL are
   provisioned automatically on deploy.

Build-time public env vars (e.g. a `VITE_PUBLIC_DOWNLOAD_URL` pointing at the R2 installers) are set
at build so the landing's download buttons point at the right place.

### API — `release-api.yml`

1. Installs deps, builds the shared `@<scope>/*` packages.
2. Deploys the Worker via `cloudflare/wrangler-action`.

## First-deploy order (one-time)

Deploy the **API first**, then the **web**. The web Worker reaches the API through a Cloudflare
**Service Binding** (`API_SERVICE` → `<app>-api`), so that Worker must already exist. After the
first deploy of each, they ship independently in any order.

## Domains

| Host                    | Serves                                              | DNS provisioning                                                          |
| ----------------------- | --------------------------------------------------- | ------------------------------------------------------------------------- |
| `<app>.example` + `www` | web Worker (landing)                                | Auto on `wrangler deploy` (custom_domain routes)                          |
| `updates.<app>.example` | R2 bucket (update feed + DMGs + `version-gate.json`)| R2 → Settings → Custom Domains                                            |
| `api.<app>.example`     | _(optional)_                                        | Only if exposing the API directly; today it's reached via Service Binding |

## Secrets & variables (GitHub `production` environment)

Deploys read from the repo's **`production`** Environment:

| Name                                                                                    | Kind     | Used by                          |
| --------------------------------------------------------------------------------------- | -------- | -------------------------------- |
| `CLOUDFLARE_API_TOKEN`                                                                  | secret   | web + api deploy (wrangler auth) |
| `CLOUDFLARE_ACCOUNT_ID`                                                                 | secret   | web + api deploy                 |
| `DATABASE_URL`                                                                          | secret   | web build + web/api Workers      |
| `BETTER_AUTH_SECRET`                                                                    | secret   | api Worker                       |
| `CORS_ORIGIN`                                                                           | variable | api Worker                       |
| `VITE_SERVER_URL`                                                                       | variable | web build + Worker               |
| `CSC_LINK`, `CSC_KEY_PASSWORD`, `APPLE_API_KEY`, `APPLE_API_KEY_ID`, `APPLE_API_ISSUER` | secrets  | desktop sign/notarize            |
| `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`                                              | secrets  | desktop → R2 upload (S3 token)   |

> If the web app proxies auth/CORS to the backend and never runs Better Auth itself, its env is only
> `DATABASE_URL` + `VITE_SERVER_URL` (+ any build-time public vars). `BETTER_AUTH_*` /
> `CORS_ORIGIN` live on the **API** only.

> **R2 credentials:** the upload uses R2's S3-compatible API, which needs **R2-specific** keys
> (`R2_ACCESS_KEY_ID` + `R2_SECRET_ACCESS_KEY`, created in **R2 → Manage R2 API Tokens**) — the
> Cloudflare API token does **not** work for it. The R2 account id reuses `CLOUDFLARE_ACCOUNT_ID`,
> and the bucket name is a plain constant in the workflow — neither is a secret.

## PR checks

Separate workflows run on PRs into `main` (none of them deploy): a cheap monorepo-wide validation
gate and per-project test suites. They are kept **separate on purpose** — a cheap universal gate vs
expensive project-specific ones. See [pr-checks.md](./pr-checks.md) and
[ci-per-project-pipelines.md](./ci-per-project-pipelines.md) for what each does and why the split.
