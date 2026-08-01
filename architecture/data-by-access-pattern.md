# Split data by access pattern (polyglot persistence, Cloudflare-flavored)

_A backend is rarely one thing. When an app mixes **live per-user data** with a
**read-massive shared catalog**, forcing both into a single reactive backend (Convex,
Firebase, etc.) overpays for the catalog and under-serves nothing. Split by access
pattern instead. Distilled from the Ripuy / trip-planner data-architecture design
(`trip-planner/docs/superpowers/specs/2026-08-01-ripuy-data-architecture.md`); the
patterns are product-agnostic._

## The core test: classify each data set

| Axis | **Live data** | **Catalog / reference data** |
| --- | --- | --- |
| Writes | constant, per-user | rare, curated by you |
| Reads | bounded, per-user | massive, repeated, shared |
| Reactivity | **needed** (real-time collab) | not needed |
| Fits | a reactive backend (Convex) | edge KV/SQL + CDN |

If two data sets land on opposite ends, **don't co-locate them**. Paying a reactive
backend's bandwidth + function-calls to serve the same static rows a thousand times is
waste; but rebuilding reactive collaboration on plain SQL is also waste. Keep each where
it's cheap.

## Pattern 1 — Read-heavy catalog on Durable Objects + SQLite

For a shared, geo-queried catalog: **one Durable Object per coarse partition** (e.g.
per country), each owning a SQLite DB. City/region/district are **indexed columns, not
shards** — geo queries cross administrative borders (a "near me" search spans
neighbourhoods), so sharding finer causes fan-out for a problem you don't have (one
SQLite holds millions of rows; your catalog has thousands). `locationHint` places each
DO near its users. The DO's SQLite **is** a durable relational store (with PITR) — not
a cache that "graduates" to Postgres. Need global analytics across partitions? Export
snapshots to a read replica (R2/warehouse); the DO stays source of truth.

## Pattern 2 — Cross-store reference = snapshot, not foreign key

When live data points at a catalog row, **don't store just an ID across two systems** —
it's a foreign key with no integrity guarantee; the catalog changes, a row is removed,
and old records break. **Snapshot the essentials at write-time** into the live store
(`{ name, geo, category, catalogId (soft ref), catalogVersion }`). The record becomes
**self-contained and offline-safe**, the catalog evolves freely, and `catalogVersion`
lets you detect drift later if it ever matters. Deliberate denormalization — accept it.

## Pattern 3 — Neutral auth issuer: one issuer, N validators

Don't let a data backend also **own identity** (e.g. Better-Auth-hosted-in-Convex makes
Convex issuer *and* validator). Every new service then needs that backend to sign its
tokens — coupling to infrastructure, worsening per service. Instead: a **standalone
issuer** (a small Better Auth Worker + its own users/sessions store). Every backend
becomes a **validator**: Convex via `customJwt` (same as adopting Clerk/Auth0), edge
stores against cached JWKS (no round-trip), WebSockets authenticated at upgrade with the
same token. Cost: you lose the reactive-session "for free" niceties → mitigate by
syncing extended user data via a webhook → a mutation in the data backend. Migrate this
**early** (cheapest at zero users; the cost grows with every service added).

## Pattern 4 — Shared package carries the contract, never the data

The cross-platform package (`catalog-core`) ships **schemas + pure logic + a
`Provider` interface**, never the dataset — embedding the JSON would bloat the web
bundle for users who never open that feature. Then per-platform providers implement the
same interface: **mobile = local SQLite** (full download once, offline queries), **web =
HTTP, lazy-loaded** on first use. The UI calls `provider.getX()` blind to what's
underneath. Cost: two providers to keep consistent → **a parity test suite** (same
fixture, identical results) is mandatory before merging changes to the shared core.

## Pattern 5 — Versioned immutable slices for passive reads

For browsing (not search), don't serve the whole catalog nor hit the compute layer.
Publish **versioned, immutable JSON slices** to object storage + CDN
(`/catalog/v12/jp-tokyo.json`), `Cache-Control: immutable, max-age=1y`. Publishing a new
version = upload `v13` + move one pointer in an `index.json`. Three read paths emerge,
each paying only for what it needs:

| Need | Route |
| --- | --- |
| Passive browsing | CDN slices — cheap, cached forever |
| Search / suggestions | compute layer (DO) on-demand — smart, low latency |
| Curation / writes | compute layer — transactional, single-writer |

## Pattern 6 — Curation as a state machine (single-writer)

Curated data ("we don't want just any data") needs states, and a single-writer store
(a DO) gives them without manual locks: `candidate → dedup → enriched → in_review →
published / rejected`. Organic growth (user templates) enters as **candidates with an
occurrence counter** — N independent mentions of the same item is a quality signal that
raises review priority. Turns growth into assisted curation instead of noise.

## When NOT to split

Splitting adds a moving part. Do it when a data set genuinely has the opposite access
pattern *and* the single-backend cost/coupling is real — not preemptively. See the
discarded-alternatives discipline in the source spec (catalog-in-Convex, catalog-in-
Postgres, D1-instead-of-DO, shard-by-city were each rejected with a reason).

## See also
- [[convex-exit-strategy]] — the same "split, don't necessarily leave" logic for Convex.
- [[repository-pattern]] · [[domain-layer-contracts]] — why swapping a store stays cheap.
- [[data-sourcing-and-seeding]] — filling the catalog with real, licensed data.
