# Infrastructure Knowledge

Product-agnostic notes on the layer **between** a deployed app and its users:
domains, DNS, edge routing and the origin-scoped configuration that a change at
this layer invalidates.

Distinct from [`monorepos/`](../monorepos/) (which covers CI/CD pipelines and
release automation) — this folder is about the runtime destination those
pipelines ship to.

## Contents

| Doc | Summary |
| --- | --- |
| [custom-domain-migration.md](./custom-domain-migration.md) | Pointing a domain at a Cloudflare Worker: imported registrar parking records block `custom_domain` deploys (522/525 + an empty `/domains/records` error), how to diagnose it with `dig`, which records must survive, apex vs `www`, and the 8-item checklist of things a new origin breaks (auth trusted origins, OAuth redirects, CORS, deep links…). |
