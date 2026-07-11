# Shared Kernel & Bounded Contexts

_Deciding what belongs in a small, stable set of shared primitives vs. what stays context-specific — with a step-by-step extraction guide._

This document covers the strategy for organizing code between shared primitives (Shared Kernel) and context-specific code (Bounded Contexts) as an application grows, plus a practical guide to extracting the shared kernel from an existing domain package.

---

## 🎯 Core Principles

### Shared Kernel

A **Shared Kernel** is a small, stable set of generic primitives shared across all bounded contexts.

- ✅ Must be **truly generic** (no context-specific logic)
- ✅ Must be **stable** (rarely changes)
- ✅ Must be **agreed upon** by all contexts
- ❌ Should be **small** (resist sharing everything)

### Bounded Context

A **Bounded Context** is a logical boundary where a particular domain model applies.

- ✅ Has its own **domain model**, **business rules**, and **repository interfaces**
- ✅ Can **use** shared kernel primitives
- ❌ Should **not** depend on other contexts directly

---

## 📊 What Belongs in the Shared Kernel

### ✅ Shared Kernel Candidates

**1. Currency** — a generic financial primitive with no business logic:

```typescript
// packages/shared-kernel/src/currency.ts
export const CURRENCIES = { USD: "USD", EUR: "EUR" } as const;
export type Currency = (typeof CURRENCIES)[keyof typeof CURRENCIES];

export const CURRENCY_INFO: Record<Currency, { label: string; symbol: string }> = {
  USD: { label: "US Dollar", symbol: "$" },
  EUR: { label: "Euro", symbol: "€" },
};
```

**2. Result Pattern** — generic error handling (like Rust's `Result`), applicable everywhere:

```typescript
// packages/shared-kernel/src/result.ts
export type Result<T, E = Error> = Success<T> | Failure<E>;
export type Success<T> = { data: T; error: null };
export type Failure<E> = { data: null; error: E };

export async function tryCatch<T, E = Error>(promise: Promise<T>): Promise<Result<T, E>> {
  try {
    const data = await promise;
    return { data, error: null };
  } catch (error) {
    return { data: null, error: error as E };
  }
}
```

**3. API Response Types** — the standard contract used by all HTTP endpoints:

```typescript
// packages/shared-kernel/src/api.ts
export interface ApiResponse<T> {
  data: T | null;
  error: { message: string } | null;
  meta?: { pagination?: PaginationMeta };
}

export interface PaginationMeta {
  page: number;
  limit: number;
  total: number;
  totalPages: number;
}

export interface PaginationParams {
  page?: number;
  limit?: number;
}
```

**4. TypeScript Utilities** — generic helpers with no business logic:

```typescript
// packages/shared-kernel/src/utils.ts
export type ObjectProperties<T> = T[keyof T];
```

### ❌ Context-Specific (Stays in the Context)

**Entity status with transitions** — contains business rules, so it stays in the context:

```typescript
// packages/your-context/domain/constants/entity-status.ts
export const ENTITY_STATUSES = {
  DRAFT: "draft", ACTIVE: "active", PAUSED: "paused",
  REJECTED: "rejected", CANCELLED: "cancelled", COMPLETED: "completed",
} as const;

// Business logic — context-specific!
export const STATUS_TRANSITIONS: Record<EntityStatus, EntityStatus[]> = {
  /* workflow rules specific to your context */
};

export function isValidStatusTransition(from: EntityStatus, to: EntityStatus): boolean {
  return STATUS_TRANSITIONS[from].includes(to);
}
```

Other examples that stay context-specific: comment types (a context-specific comment workflow), priority levels (currently context-specific — could become shared if other contexts need it).

---

## 🎯 Decision Framework

Use this to decide where code belongs:

```
Is it truly generic? (e.g., ISO standard, no business rules)
├─ Yes → shared-kernel/
└─ No → Has context-specific business rules?
    ├─ Yes → [context]/domain/
    └─ No → Used for integration between contexts?
        ├─ Yes → integration/[context]-api/
        └─ No → Keep in the context
```

### Checklist Before Adding to the Shared Kernel

- [ ] Used by **multiple contexts** (or will be soon)
- [ ] **No business logic** (pure data/utilities)
- [ ] **Stable** (won't change often)
- [ ] **Generic** (not domain-specific)
- [ ] **Small** (resist over-sharing)
- [ ] **Agreed upon** (all contexts accept it)

---

## 📦 Recommended Package Structure

### Phase 1: Single Context with Shared Kernel

```bash
packages/
  shared-kernel/          # Generic primitives
    src/{api,currency,result,utils,index}.ts
  domain/                 # Your context (still generically named)
    src/
      constants/          # entity-status, comment-type, priority-level (context-specific)
      repositories/
      schemas/
  database/               # Infrastructure (schema, repositories, mappers)
```

### Phase 2: Multiple Bounded Contexts

```bash
packages/
  shared-kernel/
  your-context/
    domain/ { constants, repositories, entities, value-objects }
    infrastructure/ { repositories, mappers }
  analytics/
    domain/ { repositories, entities }
    infrastructure/ { repositories, mappers }
  database/               # Shared DB schemas + client
```

---

## 📥 Import Strategies

**Direct imports (recommended)** — explicit about what's shared vs context-specific, best for tree-shaking:

```typescript
import { CURRENCIES, Currency } from "@app/shared-kernel/currency";
import { Result } from "@app/shared-kernel/result";
import { ENTITY_STATUSES } from "@app/domain/constants";
```

**Barrel exports (convenience)** — shorter imports, single entry point, but may pull in unused code:

```typescript
import { CURRENCIES, Result, ApiResponse } from "@app/shared-kernel";
```

**Context re-exports** — the domain package re-exports shared-kernel types for a single import surface (convenient, but hides the shared-kernel dependency):

```typescript
// packages/domain/src/index.ts
export { CURRENCIES, Currency, CURRENCY_INFO } from "@app/shared-kernel/currency";
export { Result, tryCatch } from "@app/shared-kernel/result";
export * from "./constants";
export * from "./repositories";
```

---

## 🔄 Implementation Guide: Extracting the Shared Kernel

Practical steps to extract a shared kernel from an existing `@app/domain` package.

### Prerequisites

- Existing `@app/domain` package
- Files to move identified
- All tests passing, clean git working directory
- Do this in a feature branch and commit incrementally

### Goal

Transform this:

```bash
packages/domain/src/
  constants/{currency,entity-status,comment-type}.ts
  types/{api-response,result}.ts
```

into this:

```bash
packages/shared-kernel/src/{api,currency,result,utils,index}.ts   # extracted
packages/domain/src/constants/{entity-status,comment-type,priority-level}.ts  # stays
```

### Step 1: Create the Shared Kernel Package

```json
// packages/shared-kernel/package.json
{
  "name": "@app/shared-kernel",
  "version": "1.0.0",
  "type": "module",
  "exports": {
    ".": "./src/index.ts",
    "./api": "./src/api.ts",
    "./currency": "./src/currency.ts",
    "./result": "./src/result.ts",
    "./utils": "./src/utils.ts"
  },
  "publishConfig": {
    "exports": {
      ".": "./dist/index.mjs",
      "./api": "./dist/api.mjs",
      "./currency": "./dist/currency.mjs",
      "./result": "./dist/result.mjs",
      "./utils": "./dist/utils.mjs"
    }
  },
  "scripts": { "build": "tsdown", "check-types": "tsc --noEmit" },
  "dependencies": { "zod": "catalog:" }
}
```

### Step 2: Create the Source Files

`src/api.ts` (ApiResponse, PaginationMeta, PaginationParams, PaginationResult), `src/currency.ts`, `src/utils.ts` (`ObjectProperties`), and `src/result.ts` with helpers:

```typescript
// packages/shared-kernel/src/result.ts (extended helpers)
export type Success<T> = { data: T; error: null };
export type Failure<E> = { data: null; error: E };
export type Result<T, E = Error> = Success<T> | Failure<E>;

export async function tryCatch<T, E = Error>(promise: Promise<T>): Promise<Result<T, E>> {
  try {
    return { data: await promise, error: null };
  } catch (error) {
    return { data: null, error: error as E };
  }
}

export function isSuccess<T, E>(r: Result<T, E>): r is Success<T> { return r.error === null; }
export function isFailure<T, E>(r: Result<T, E>): r is Failure<E> { return r.error !== null; }

/** Unwrap or throw — use only when you're certain it's a success. */
export function unwrap<T, E>(r: Result<T, E>): T {
  if (r.error) throw r.error;
  return r.data as T;
}
```

### Step 3: Barrel Export

```typescript
// packages/shared-kernel/src/index.ts
export type { ApiResponse, PaginationMeta, PaginationParams, PaginationResult } from "./api";
export { CURRENCIES, CURRENCY_VALUES, CURRENCY_INFO, isValidCurrency } from "./currency";
export type { Currency } from "./currency";
export { tryCatch, isSuccess, isFailure, unwrap } from "./result";
export type { Result, Success, Failure } from "./result";
export type { ObjectProperties } from "./utils";
```

### Step 4: Re-export from Domain (Backward Compatibility)

```typescript
// packages/domain/src/types/index.ts
export type { ApiResponse, PaginationMeta, PaginationParams, Result, Success, Failure, ObjectProperties } from "@app/shared-kernel";
export { tryCatch, isSuccess, isFailure, unwrap } from "@app/shared-kernel";
```

Existing imports from `@app/domain` keep working — migrate call sites to `@app/shared-kernel/*` gradually.

### Step 5: Wire Dependencies and Verify

Add `"@app/shared-kernel": "workspace:*"` to every package/app that needs it (`domain`, `database`, `apps/api`, `apps/web`), then:

```bash
bun install
bun run build          # build shared-kernel, then domain
bun run check-types
bun test
```

Verification checklist: install runs clean, all packages build, type-check passes, no import errors, tests pass, dev and prod builds work.

### Common Issues

- **Circular dependency:** the shared kernel must NEVER import from the domain. Only the domain imports from the shared kernel.
- **Types not found after migration:** run `bun install`, confirm `node_modules/@app/shared-kernel` exists, restart the TypeScript server.

---

## 💡 Key Takeaways

1. **Shared Kernel = Small + Stable + Generic** — only primitives with no business logic (Currency, Result, API types).
2. **Context-Specific = Business Rules + Domain Logic** — status transitions, comment workflows, repository interfaces.
3. **Start small, refactor as needed** — begin with `@app/domain`, extract the shared kernel when it's clear, split into contexts when you add a second domain.
4. **Resist over-sharing** — don't share just because code looks similar; duplicate intentionally when meanings diverge; keep contexts independent.

## 🔗 Related

- [Bounded Contexts Complete Guide](./bounded-contexts-complete-guide.md)
- [Repository Pattern](./repository-pattern.md)
- [Domain Architecture Patterns](./domain-architecture-patterns.md)
