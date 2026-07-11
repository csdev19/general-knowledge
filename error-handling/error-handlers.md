# Error Handlers

_Utilities for handling Result types and turning them into consistent errors._

## Overview

Error handler utilities provide consistent error handling patterns for database operations and other async operations that use the Result type pattern.

## Why Use Error Handlers?

1. **Consistency**: All errors are handled the same way across the application
2. **Type Safety**: TypeScript ensures all error cases are handled
3. **DRY**: Avoid repeating error handling logic in every route handler
4. **Maintainability**: Centralized error handling makes it easier to update error logic

## Handler Functions

### `handleDatabaseResult<T>(result: Result<T[], Error>, notFoundMessage?: string): T`

Handles a database query Result, throwing appropriate errors. Expects a single item.

```typescript
import { tryCatch } from "@your-scope/domain/types";
import { handleDatabaseResult } from "../utils/error-handlers";

const result = await tryCatch(
  db.select().from(entities).where(eq(entities.id, entityId)).limit(1)
);

const entity = handleDatabaseResult(result, "Entity not found");
// entity is guaranteed to exist here
```

### `handleSingleDatabaseResult<T>(result: Result<T[], Error>, notFoundMessage?: string): T`

Alias for `handleDatabaseResult`. Handles queries that should return exactly one item.

```typescript
const entity = handleSingleDatabaseResult(result, "Entity not found");
```

### `handleListDatabaseResult<T>(result: Result<T[], Error>): T[]`

Handles a database query Result that expects multiple items. Returns empty array if no results (doesn't throw).

```typescript
const result = await tryCatch(
  db.select().from(entities).where(eq(entities.status, "active"))
);

const entities = handleListDatabaseResult(result);
// entities is an array (may be empty)
```

### `handleMutationResult<T>(result: Result<T[], Error>, notFoundMessage?: string): T`

Handles a database mutation Result (insert, update, delete). Throws error if no rows affected.

```typescript
const result = await tryCatch(
  db.update(entities)
    .set({ name: "New name" })
    .where(eq(entities.id, entityId))
    .returning()
);

const updated = handleMutationResult(result, "Entity not found");
```

### `handleResult<T, E>(result: Result<T, E>, errorMapper?: (error: E) => Error): T`

Generic handler with custom error mapping. Useful for non-database operations.

```typescript
const result = await tryCatch(validateInput(input));

const validInput = handleResult(result, (error) => {
  if (error instanceof ValidationError) {
    return new BadRequestError(error.message);
  }
  return new InternalServerError("Validation failed");
});
```

## Usage Examples

### Single Item Query

```typescript
.get("/:id", async ({ params }) => {
  const result = await tryCatch(
    db.select()
      .from(entities)
      .where(eq(entities.id, params.id))
      .limit(1)
  );

  const entity = handleDatabaseResult(result, "Entity not found");
  return successBody(entity);
})
```

### List Query

```typescript
.get("/", async () => {
  const result = await tryCatch(
    db.select().from(entities).where(eq(entities.status, "active"))
  );

  const entities = handleListDatabaseResult(result);
  return successBody(entities);
})
```

### Update Mutation

```typescript
.put("/:id", async ({ params, body }) => {
  const result = await tryCatch(
    db.update(entities)
      .set(body)
      .where(eq(entities.id, params.id))
      .returning()
  );

  const updated = handleMutationResult(result, "Entity not found");
  return successBody(updated);
})
```

### Custom Error Mapping

```typescript
const result = await tryCatch(complexOperation());

const data = handleResult(result, (error) => {
  if (error instanceof CustomError) {
    return new BadRequestError(error.message);
  }
  return new InternalServerError("Operation failed");
});
```

## Error Types

These handlers throw domain-specific errors:

- `NotFoundError` — When resource is not found
- `InternalServerError` — When database operations fail
- `BadRequestError` — When validation fails (with custom mapper)

## Best Practices

1. **Use appropriate handler**: Choose the handler that matches your operation type
2. **Provide meaningful messages**: Customize `notFoundMessage` for better error messages
3. **Combine with Result types**: Always use `tryCatch` first, then handle the result
4. **Let the framework handle errors**: These handlers throw errors that the backend's global error handler will catch

## See Also

- [Result Types](./result-types.md) — Understanding the Result type pattern
- [Response Helpers](./response-helpers.md) — Creating error responses
