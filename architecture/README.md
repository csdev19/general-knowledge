# Architecture Knowledge Hub

A generalized, product-agnostic collection of architecture patterns for building monorepo applications with Domain-Driven Design and Hexagonal Architecture. Every doc keeps the pattern and the reasoning; product-specific names have been stripped in favor of generic domain terms.

The core stack these patterns assume: a TypeScript monorepo with a `domain` / `application` / `infra` layering, a shared UI package, a web app (server functions) and an HTTP API as consumers.

---

## Core DDD & Layering

Start here to understand how the packages are organized and how the layers depend on each other.

- **[application-layer.md](./application-layer.md)** — Layer-first package architecture; why business logic lives in a shared `application` package that multiple consumers wire to their own infra.
- **[application-services-layer.md](./application-services-layer.md)** — Implementing use cases with CQRS commands/queries, DTOs, mappers, and application services over a rich domain.
- **[repository-pattern.md](./repository-pattern.md)** — Splitting a repository into a domain interface and an infra implementation, with a before/after refactoring guide.
- **[bounded-contexts-complete-guide.md](./bounded-contexts-complete-guide.md)** — Full DDD per bounded context: layers, context integration (DTOs, events, ACL), and a single-to-many migration path.
- **[shared-kernel.md](./shared-kernel.md)** — What belongs in a small stable set of shared primitives vs. context-specific code, plus a step-by-step extraction guide.

## Domain Modeling

How to decide how much modeling machinery a domain actually needs.

- **[domain-architecture-patterns.md](./domain-architecture-patterns.md)** — Entity-based (DDD) vs. schema-based modeling: trade-offs, when to use each, and a gradual migration path.
- **[domain-modeling-strategy.md](./domain-modeling-strategy.md)** — A pragmatic middle path: contract types as wire truth, branded IDs, value objects only for high-leverage primitives, invariants in use cases.
- **[domain-layer-contracts.md](./domain-layer-contracts.md)** — Extracting repository interfaces and plain entity types into the domain package so infra implements domain-defined contracts.

## Data & backend topology

- **[data-by-access-pattern.md](./data-by-access-pattern.md)** — Split a backend by access pattern instead of forcing one: live/reactive data vs read-massive catalog; Durable Objects + SQLite for a geo catalog (partition by country, not city); snapshot-not-FK cross-store refs; a neutral auth issuer (one issuer, N validators); shared package carries contract-not-data; versioned immutable CDN slices; curation as a single-writer state machine.

## Cross-Cutting Concerns

Patterns that span every layer and consumer.

- **[security-hardening.md](./security-hardening.md)** — Reusable patterns for DB-level authorization, soft delete, N+1 elimination, transactional writes, and invite enforcement.
- **[observability.md](./observability.md)** — Structured error logging, sanitization rules, and a maturity roadmap so users never see internal errors.

## Decisions (ADRs)

- **[decisions/adr-0001-effect-use-cases.md](./decisions/adr-0001-effect-use-cases.md)** — Adopting Effect-TS for application use cases to solve dependency injection, typed error channels, and dual-consumer wiring. For the adoption decision itself and the writing patterns, see [effect/](../effect/).

---

## Suggested Reading Order

1. **application-layer** → **repository-pattern** — the structural backbone.
2. **domain-architecture-patterns** → **domain-modeling-strategy** — decide how rich your domain should be.
3. **shared-kernel** → **bounded-contexts-complete-guide** — scale from one domain to many.
4. **security-hardening** + **observability** — harden and instrument whatever you built.
5. **decisions/** — the reasoning behind specific technology choices.
