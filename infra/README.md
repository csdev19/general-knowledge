# Infrastructure Knowledge

Product-agnostic notes on the layer **between** a deployed app and its users:
domains, DNS, edge routing, secrets management and the origin-scoped
configuration that a change at this layer invalidates.

Distinct from [`monorepos/`](../monorepos/) (which covers CI/CD pipelines and
release automation) — this folder is about the runtime destination those
pipelines ship to.

## Contents

| Doc                                                        | Summary                                                                                                                                                                                                                                                                                                                                                |
| ---------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| [custom-domain-migration.md](./custom-domain-migration.md) | Pointing a domain at a Cloudflare Worker: imported registrar parking records block `custom_domain` deploys (522/525 + an empty `/domains/records` error), how to diagnose it with `dig`, which records must survive, apex vs `www`, and the 8-item checklist of things a new origin breaks (auth trusted origins, OAuth redirects, CORS, deep links…). |
| [infisical-secrets.md](./infisical-secrets.md)             | **The accepted secrets playbook, independent of Varlock.** Folders by risk, tags by consumer, selection versus authorization, scoped process/Worker delivery, build and signing boundaries, precedence, cache inputs, and explicit local/production migration status.                                                                                  |
| [environment-inventory.md](./environment-inventory.md)     | **README → project inventory → shared playbook.** Reusable documentation contract and template: every key's consumer, folder/tag, requirement, default, source, read stage, propagation, and evidence; includes optional/test/release values without duplicating project inventories in this hub.                                                      |
| [varlock-evaluation.md](./varlock-evaluation.md)           | varlock as a declaration layer on top of Infisical: what it adds, and the three reasons it is **not** the default — an existing zod env module already provides the schema, Cloudflare needs two different integrations, and the syntax outruns its docs. Records what was verified against 1.19.0, and when it is still worth reaching for.           |

## Email

- [Transactional email for Niway apps](./transactional-email.md) — shared Resend sender domain, app-specific addresses, scope and the correction to earlier migration advice.
