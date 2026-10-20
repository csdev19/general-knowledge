# App names

_One name per app, used everywhere the app shows up: its directory, its deployed resource and its
secrets tag. Detail goes after the name, never before it._

## The convention

Every app a product ships gets one name, `<product>-<role>`:

```text
<product>-api        the API
<product>-web        the website / web app
<product>-desktop    the desktop app
```

That same string is:

| Where | Example |
| --- | --- |
| The app directory | `apps/<product>-api` |
| The workspace package name | `<product>-api` |
| The deployed resource (Worker, service, container) | Worker `<product>-api` |
| The base tag of its secrets in the secrets store | tag `<product>-api` |

**The role says what the app does, not what it is built with.** `web-hono` and `server-hono`
name a framework; a reader cannot tell which one serves the site. `<product>-web` and
`<product>-api` can be read without opening either.

**The product prefix stays even inside the product's own repository.** Deployed resources and
secrets tags are seen next to other products' ones (one Cloudflare account, one secrets
organisation), and a directory named like its Worker removes a lookup.

## Detail goes after the base name

When one app has more than one consumer of configuration, each extra consumer is a child of the
app's name: `<app>-<detail>`.

```text
<product>-desktop            values the desktop build reads
<product>-desktop-sign       signing and notarization, read only by the release
<product>-desktop-publish    the update-bucket upload, read only by the release
<product>-api-db             database tools: schema push, migrations, seeds
<product>-api-deploy         credentials to deploy the API
```

Sorting by name then groups everything that belongs to one app, and the base name alone answers
"which app breaks if this changes?".

A child is justified only when its consumer holds a **different authority** from the app itself.
Signing keys are the clearest case: the release signs, an ordinary desktop development run must
never receive them. A child that would receive exactly the app's values is not a child; reuse the
base tag.

## What never goes in the name

- **The environment.** `dev` and `prod` are environments of the secrets store and of the
  deployment target. The tag `<product>-api` selects the dev value on a laptop and the prod value
  in a release.
- **The framework or the runtime stage** (`-hono`, `-runtime`). The first says how, the second is
  implied by the base name.
- **A combined consumer.** A value two apps need gets two tags (`<product>-api-deploy` and
  `<product>-web-deploy`), not a tag naming both.

## Adopting it in an existing repository

- **Tags first.** They cost nothing to rename before the secrets store is in use.
- **Deployed resources next.** Renaming a Worker creates a new one: secrets are re-uploaded, a
  custom domain must be detached from the old one, and the old one is deleted by hand.
- **Directories last, when convenient.** A directory rename touches release configuration
  (release-please package paths), CI path filters and every doc link; schedule it rather than
  doing it in passing.

## Measured example

Yui (`invisible-assistant`, 2026-10-20): the Workers are already `yui-api` and `yui-web`; the
directories are still `apps/api` and `apps/web` (scheduled). Secrets tags:

| Tag | Consumer |
| --- | --- |
| `yui-api` | The API Worker and its local `.env` |
| `yui-api-db` | `db:*` scripts and the dev seed |
| `yui-api-deploy` | The deploy workflow, for the API Worker |
| `yui-web` | The web Worker, its build and its local `.env` |
| `yui-web-deploy` | The deploy workflow, for the web Worker |
| `yui-desktop` | The desktop build |
| `yui-desktop-sign` | Signing and notarization in the release |
| `yui-desktop-publish` | The upload to the update bucket |

Kaipu predates this page: its tags (`server-hono`, `web-hono`, `kaipu-record`, `db-scripts`,
`ci-deploy`) name frameworks or groups, and its signing values share the desktop tag. Under this
convention they become `kaipu-api`, `kaipu-web`, `kaipu-desktop`, `kaipu-api-db`,
`kaipu-desktop-sign`, and so on. Not migrated yet.

## Related

- [Infisical secrets](../infra/infisical-secrets.md) — folders by what a leak costs, tags by
  consumer; this page names the tags.
- [App bundle identifiers](./app-bundle-identifiers.md) — the OS-level identity
  (`dev.niway.<product>`), which follows its own rule and is frozen at first release.
