# Security Hardening

_Reusable patterns for closing authorization, data-safety, N+1, and validation gaps: DB-level authorization, soft delete, batch queries, transactional writes, and invite enforcement._

A code review that surfaces authorization, data-safety, performance, and validation issues tends to produce the same handful of reusable patterns. This doc captures those patterns so they can be applied proactively rather than rediscovered.

## Patterns

### Authorization at the DB query level

Server functions and API handlers that mutate user-owned resources must verify ownership **at the DB query level**, not just check that the caller is authenticated. Scoping the query is the single source of truth and avoids TOCTOU (time-of-check to time-of-use) race conditions.

- Scope every mutating query with the owner: `UPDATE ... WHERE id = ? AND userId = ?`.
- For relationship-based access, verify the caller is a party to the record — e.g. creating a settlement checks the caller's `membership.id` is `fromMemberId` or `toMemberId`; a "mark as paid" action verifies the caller is the creditor (`toMemberId`).

```typescript
// ❌ Race-prone: check in the handler, then call an unscoped repo
if (notification.userId !== caller.id) throw forbidden();
await repo.markAsRead(notificationId); // unscoped UPDATE

// ✅ Authorization is part of the query — impossible to bypass
await repo.markAsRead(notificationId, caller.id); // UPDATE ... WHERE id = ? AND userId = ?
```

> ⚠️ Add the owner parameter to the **repository interface signature** (e.g. `markAsRead(id, userId)`) so every implementation is forced to scope the query.

### Soft delete

Deletion sets a `deletedAt` timestamp instead of removing the row. Every read and every join that touches the table must exclude soft-deleted rows.

- `delete()` sets `deletedAt = new Date()`.
- Every query that reads the entity (`findById`, `findByGroupId`, `getStats`) filters `deletedAt IS NULL`.
- Every query that **joins** on the entity (e.g. a balance query joining line items) filters `deletedAt IS NULL` on the joined table.

> ⚠️ When you add a new query that reads the entity, you **must** include the `isNull(table.deletedAt)` filter — forgetting it silently includes deleted rows. For joins, the filter must be on the **joined** table, not a separate subquery.

### Batch queries (N+1 elimination)

Replace per-row loops with a single `WHERE ... IN (...)` query, then group results into a `Map` at the call site.

```typescript
// ❌ N+1 — one query per parent
for (const item of items) {
  item.children = await repo.getChildren(item.id);
}

// ✅ 2 queries total regardless of item count
const children = await repo.getChildrenForItems(items.map((i) => i.id)); // WHERE item_id IN (...)
const byItem = Map.groupBy(children, (c) => c.itemId);
```

> ⚠️ Batch methods return **flat arrays** with a parent-id field. The caller must group into Maps — don't assume the array is ordered by parent.

Prefer this over a dataloader abstraction for an MVP, and over denormalized JOINs that return duplicated rows and are harder to map.

### Transactional multi-step writes

When an update involves delete-then-insert of dependent rows, wrap it in a transaction so a crash between steps can't lose data.

```typescript
// Update parent + replace dependents atomically
await db.transaction(async (tx) => {
  await tx.update(parentTable).set(data).where(eq(parentTable.id, id));
  await tx.delete(childTable).where(eq(childTable.parentId, id));
  await tx.insert(childTable).values(newChildren);
});
```

> ⚠️ HTTP-based database drivers can still support transactions via an atomic HTTP batch API — don't assume "HTTP driver = no transactions."

### Invite / share-link enforcement

Enforce all limits at the **query level** so no caller can bypass them, rather than fetching the record and checking in application code (race-prone).

```typescript
// findActiveByCode enforces every limit in one query
WHERE deactivatedAt IS NULL
  AND (expiresAt IS NULL OR expiresAt >= now())
  AND (maxUses  IS NULL OR useCount < maxUses)
```

Apply the same pattern to expiring share links (`findBySlug` checks `expiresAt`).

### Lifecycle guards must be applied everywhere

If an action rejects records in a certain state (e.g. a member that has `leftAt` set cannot be claimed), the **preview** of that action needs the identical guard.

> ⚠️ A "preview" (`getClaimPreview`) and its "commit" (`claimMember`) must share the same guards. It's easy to fix one and forget the other.

## Decisions

| Decision                                    | Why                                                            | Alternatives Considered                                      |
| ------------------------------------------- | ------------------------------------------------------------- | ------------------------------------------------------------ |
| Scope authorization at the DB query level   | Prevents TOCTOU races; single source of truth                 | Check in the handler, then call an unscoped repo (race-prone) |
| Batch queries with an `IN` clause           | Reduces N+1 to a constant 2 queries regardless of row count   | Dataloader (over-engineering for MVP); denormalized JOINs    |
| Wrap delete+insert in a transaction         | Ensures atomicity of dependent-row replacement                | Sequential queries (a crash between steps loses data)        |
| Enforce invite limits at the query level    | Single check point; impossible to bypass from any caller      | Check after fetching (race between check and use)            |

## Verification Checklist

When reviewing or testing these areas:

1. Mark a resource as read/updated → verify only the caller's own rows are affected.
2. Delete a record → verify it's soft-deleted (still in DB, excluded from UI and from any derived/aggregate queries).
3. Create a relationship record → verify it's rejected if the caller isn't a party.
4. Perform a creditor/owner-only action → verify it's rejected for non-owners.
5. Use an expired/maxed-out invite → verify it's rejected.
6. Open an expired share link → verify "not found."

## Related

- [Observability & Error Handling](./observability.md) — Authorization errors should log server-side and return generic messages to the client.
- [Repository Pattern](./repository-pattern.md) — Where these query-level guards live.
