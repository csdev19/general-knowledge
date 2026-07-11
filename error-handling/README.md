# Error Handling

A knowledge hub for consistent, type-safe error handling in a TypeScript backend: from the functional `Result` type up to a centralized, production-safe error-handling wrapper.

Read the docs in this order:

1. **[Result Types](./result-types.md)** — Functional, type-safe error handling with a Rust-inspired `Result<T, E>` discriminated union and its helper functions (`tryCatch`, `isSuccess`, `isFailure`, `unwrap`).
2. **[Response Helpers](./response-helpers.md)** — Helper functions (`successBody`, `createdBody`, `errorBody`, pagination) that produce standardized API responses with the right HTTP status codes.
3. **[API Response Types](./api-response-types.md)** — The `ApiResponse<T>` envelope shape used by every endpoint, including error and pagination metadata.
4. **[Error Handlers](./error-handlers.md)** — Utilities that turn `Result` values into consistent thrown errors (`handleDatabaseResult`, `handleListDatabaseResult`, `handleMutationResult`, `handleResult`).
5. **[Error Handling Overhaul](./error-handling-overhaul.md)** — A real-world retrospective: a centralized `withErrorHandling` wrapper, typed `AppError` errors, and structured JSON logging that keeps internal error details out of user-facing responses.
