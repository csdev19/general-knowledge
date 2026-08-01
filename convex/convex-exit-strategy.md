# Convex exit strategy — when & how to move to a cheaper, more resilient backend

_A ready-to-pull migration step, captured before it is needed. Convex is excellent
for building fast (reactive queries, transactional mutations, hosted auth, zero-ops),
but its cost curve and vendor lock-in eventually justify moving. This is the plan for
that day — the trigger, the target options, the migration surface, and what we already
did that de-risks it. **Status: not decided; documented so it is a clear step, not a
scramble.**_

## Why you'd leave Convex (the triggers)

Pull this plan when one or more of these bite:

- **Cost inflection.** Convex bills on function calls + bandwidth + storage; a
  read-heavy reactive app (every `useQuery` is a live subscription) can climb fast.
  Watch the usage dashboard; the migration ROI appears when the monthly bill exceeds a
  managed Postgres (Neon) + Workers setup by a comfortable margin.
- **Resilience / vendor lock-in.** One vendor for data + auth + hosting is a single
  point of failure and a single pricing lever. A portable Postgres stack is insurable.
- **Admin-access friction** (already hit building Ripuy): Convex `internal*` functions
  are **not** callable from `ConvexHttpClient` — only via the `convex` CLI / deploy
  keys. Operating Convex from outside the app (seeding, back-office, batch jobs) is
  awkward by design.

## What Convex gives you (so you know what you're replacing)

| Capability | Convex | Replacement cost when you leave |
| --- | --- | --- |
| Reactive queries (`useQuery` live) | built-in | **the hard part** — rebuild with polling / SSE / WebSockets or TanStack Query + invalidation |
| Transactional mutations | built-in | Postgres transactions (Drizzle) |
| Hosted auth | Better Auth **in** Convex | Better Auth + `drizzleAdapter` (the rakoi pattern already does this) |
| Typed generated API | `_generated` | oRPC / Hono contracts (`api-contract` package) |
| File storage | built-in | R2 / S3 |
| Zero-ops hosting | built-in | Cloudflare Workers + Neon (still low-ops) |

## Target options

1. **The rakoi stack (recommended default).** Neon (serverless Postgres) + Drizzle +
   Cloudflare Workers + Hono/oRPC + Better Auth (`drizzleAdapter`, scrypt). Cristian's
   standard for new NIWAY apps — portable, cheap at scale, and the team already knows it.
   Trade: you rebuild reactivity and own a bit more infra. See [[fullstack-hono-orpc]].
2. **Managed Postgres with realtime** (Supabase / Postgres + PostgREST). Keeps a
   realtime channel out of the box; less rebuild of the live layer, still Postgres-portable.
3. **Self-hosted Postgres** (mini or a cheap VPS). Max control, min cost, most ops —
   only if the workload is small and you want zero vendor bill.

## Migration surface (Convex-specific → portable)

- **Schema:** `convex/schema.ts` (`defineTable` + `validators.ts`) → Drizzle schema.
  Mechanical; the shapes are already explicit.
- **Functions:** `query` / `mutation` / `action` / `internal*` → API endpoints
  (oRPC/Hono) backed by a repository layer. **Big win if the app already uses the
  domain ← application ← infra split** ([[repository-pattern]], [[domain-layer-contracts]]):
  those layers are framework-agnostic, so only the thin Convex adapter is rewritten.
- **Reactivity:** `useQuery` live subscriptions → TanStack Query with invalidation, or
  SSE/WebSockets for true push. Budget the most time here — it is the one thing Convex
  gives for free that Postgres does not.
- **Auth:** Better-Auth-in-Convex → Better Auth + `drizzleAdapter` (rakoi already ships this).
- **Data:** `convex export` (JSONL) → transform → load into Postgres. **The Ripuy
  `db export`/`import` commands are exactly this bridge** — see [[ai-agent-delegation]]
  and the Ripuy CLI (`packages/cli` in trip-planner).

## What we already did that de-risks this

- **Framework-agnostic domain/application layers** → the swap touches infra only.
- **Ripuy CLI + `ripuy schema --json` + JSONL export/import** → a stable operational
  interface and a data-portability bridge that both survive the backend change.
- **Tiered CI + the `verify` gate** ([[ci-cd-pipeline-strategy]]) → independent of the
  backend, so the migration branch is validated the same way.

## Do these BEFORE a full migration (cheaper wins first)

Don't migrate to save money before exhausting the easy levers: review the Convex usage
dashboard, add indexes, paginate list queries, cache derived reads, and cut needless
reactive subscriptions. If only a few tables drive the cost, consider a **strangler**
migration (move those tables/endpoints first) instead of a big-bang rewrite.

## Open decisions (resolve when you pull the trigger)

- Target stack: rakoi (Neon+Drizzle+Workers) vs managed-realtime (Supabase) vs self-host.
- How to replace reactivity: TanStack Query polling vs SSE/WS.
- Big-bang vs strangler (table-by-table).
- Timing (tie to a cost threshold, not a whim).

## See also
- [[fullstack-convex]] — building **on** Convex (the thing you're migrating from).
- [[fullstack-hono-orpc]] — the likely target shape.
- [[repository-pattern]] · [[domain-layer-contracts]] — why the swap stays cheap.
- [[ci-cd-pipeline-strategy]] — the gate that validates the migration branch.
