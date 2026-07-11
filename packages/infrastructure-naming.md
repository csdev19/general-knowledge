# Infrastructure Package Naming Strategy

_Naming convention and structural intent for infrastructure packages in a monorepo — the `infra-<capability>` convention and why it works._

## Why Change the Naming?

Early in a project, renaming infra packages has **low cost** and **high long-term payoff**.

Goals:

- Make architectural intent obvious
- Prevent domain/application pollution
- Scale cleanly as more infrastructure adapters are added
- Avoid technology-coupled naming

---

## Core Principle

> **Infrastructure packages represent adapters, not technologies.**

They implement contracts defined in the **domain** or **application**, and are wired together by the **server**.

---

## Naming Convention

### Use: `infra-<capability>`

Examples:

- `@scope/infra-db`
- `@scope/infra-auth`
- `@scope/infra-email` (future)
- `@scope/infra-notifications` (future)

Where `<capability>` describes **what the package does**, not **how it does it**.

---

## Why `infra-*` Works Well

### 1. Explicit Architectural Intent

- Makes it immediately clear this is **infrastructure**
- Discourages accidental imports into domain or application
- Reinforces dependency direction: infra depends on domain, never the reverse

### 2. Scales Cleanly

- Adding `infra-email`, `infra-notifications`, `infra-storage` follows the same pattern
- All infra packages are visually grouped in the file tree under `packages/infra-*`

### 3. Technology-Agnostic

- `infra-db` could be backed by Drizzle + Neon today, Prisma + Supabase tomorrow
- The package name describes the **capability** (database access), not the **implementation** (Drizzle)

### 4. Import Clarity

- When you see `@scope/infra-db` in an import, you immediately know it's infrastructure
- Makes code review easier: infra imports in `domain/` or `application/` are instant red flags

---

## Example Infrastructure Packages

| Package             | Capability      | Implements                                 |
| ------------------- | --------------- | ------------------------------------------ |
| `@scope/infra-db`   | Database access | Drizzle ORM repositories, mappers, schemas |
| `@scope/infra-auth` | Authentication  | Better Auth server configuration           |
