# Convex Knowledge Hub

Patterns for building on **Convex** as a reactive backend — the client connection and
hosting **Better Auth inside Convex**. Complements the [fullstack-convex stack
recipe](../stacks/fullstack-convex.md), which composes these into a build guide.

> Scope: architecture / integration patterns learned in real projects. For low-level
> Convex function patterns (queries, mutations, actions, migrations, cron, file storage),
> use the `convex-*` dev-environment skills.

## Contents

| File | Summary |
| --- | --- |
| [client-connection.md](./client-connection.md) | `ConvexReactClient`, providers, the two URLs (`.cloud` vs `.site`), reactive `useQuery`, consuming the generated API in a monorepo, first-time deployment. |
| [convex-exit-strategy.md](./convex-exit-strategy.md) | When & how to move **off** Convex to a cheaper, more resilient backend (Neon+Drizzle+Workers / managed PG): triggers, what you're replacing (reactivity is the hard part), the migration surface, and what already de-risks it (framework-agnostic layers + Ripuy JSONL export/import). A ready step, not a decision. |
| [module-paths.md](./module-paths.md) | Why a hyphen in any file name under `convex/` (including `_lib/` and `__tests__/`) fails the first deployment push while type-check and tests stay green, the camelCase rule, and a credential-free guard test. |
| [better-auth.md](./better-auth.md) | Better Auth hosted **in** Convex (`@convex-dev/better-auth`): the full server setup, the SDK 57 / React 19 version constraints + the `useSession` hang fix, `expo-network`, transitive-dep overrides, the Expo cast, `exp://` trusted origins, and an auth-hang diagnostic playbook. |
