# Infrastructure Knowledge

Product-agnostic notes on the layer **between** a deployed app and its users:
domains, DNS, edge routing, secrets management and the origin-scoped
configuration that a change at this layer invalidates.

Distinct from [`monorepos/`](../monorepos/) (which covers CI/CD pipelines and
release automation) — this folder is about the runtime destination those
pipelines ship to.

## Contents

| Doc | Summary |
| --- | --- |
| [custom-domain-migration.md](./custom-domain-migration.md) | Pointing a domain at a Cloudflare Worker: imported registrar parking records block `custom_domain` deploys (522/525 + an empty `/domains/records` error), how to diagnose it with `dig`, which records must survive, apex vs `www`, and the 8-item checklist of things a new origin breaks (auth trusted origins, OAuth redirects, CORS, deep links…). |
| [infisical-varlock-secrets.md](./infisical-varlock-secrets.md) | Secrets/env playbook: Infisical as the single value store, committed varlock `.env.schema` files as the declaration layer, machine identities (Universal Auth locally, OIDC in CI so GitHub Secrets end up empty), secret-path layout by consumer, Cloudflare `varlock-wrangler` sharp edges, migration order for an existing repo, and the traps found in the first adoption. |

## Email

- [Transactional email for Niway apps](./transactional-email.md) — shared Resend sender domain, app-specific addresses, scope and the correction to earlier migration advice.
