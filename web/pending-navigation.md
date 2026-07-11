# Pending Navigation Indicator

_GlobalPendingIndicator + per-route `pendingComponent` pattern for route transition feedback._

## Overview

Two complementary patterns provide loading feedback during navigation:

1. **GlobalPendingIndicator** — a thin accent bar at the top of every page, visible on any route transition
2. **Per-route `pendingComponent`** — a skeleton rendered while a specific route's loader is running

## Files

| Path                                              | Purpose                                             |
| ------------------------------------------------- | --------------------------------------------------- |
| `apps/web/src/index.css`                          | `@keyframes loading-bar` animation                  |
| `apps/web/src/routes/__root.tsx`                  | `GlobalPendingIndicator` component + render site    |
| `apps/web/src/routes/_authenticated/.../index.tsx` | Example route with `pendingComponent` + `pendingMs` |

## How It Works

### Global bar

`GlobalPendingIndicator` uses `useRouterState({ select: (s) => s.isLoading })` to detect any in-flight navigation. When loading, it renders a fixed `h-0.5` div at `z-[60]` (above the header's `z-50`) with a half-width child that translates across using `loading-bar` keyframes. Returns `null` when not loading — no DOM cost at rest.

The component is rendered inside the app's error boundary but outside the page layout `div`, so it doesn't affect document flow.

### Per-route skeleton

Routes that use `ensureQueryData` in their loader block navigation until data is ready. Adding `pendingComponent` and `pendingMs` to the route definition shows a skeleton if the loader takes longer than `pendingMs` milliseconds.

```
pendingMs: 0   → show skeleton immediately (no delay)
pendingMs: 200 → wait 200ms before showing (avoids flash on fast loads)
```

A detail route whose SSR loader blocks typically uses `pendingMs: 0` because users expect immediate feedback on slow connections.

## Decisions

| Decision                        | Why                                                                                                         | Alternatives Considered                                                           |
| ------------------------------- | ----------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------- |
| `z-[60]` for indicator          | Must appear above fixed header (`z-50`)                                                                     | Same z-index as header (indicator hidden behind header)                           |
| `bg-accent` bar color           | Matches brand; consistent with interactive accent usage                                                     | White or neutral (less branded)                                                   |
| `pendingMs: 0` for detail route | Loader blocks navigation — immediate feedback is always correct                                             | `pendingMs: 200` (creates 200ms blank gap on slow loads)                          |
| Keyframe in `index.css`         | Must be in global CSS scope; Tailwind `animate-*` utilities can't reference custom keyframes without config | `tailwind.config.ts` animation key (no config file with Tailwind v4)              |

## Gotchas & Warnings

- ⚠️ `useRouterState` must be imported from `@tanstack/react-router` — it's on the same import line as `HeadContent`, `Outlet`, `Scripts`, etc. Easy to miss when adding to `__root.tsx`.
- ⚠️ The `loading-bar` keyframe must live in `index.css` (global CSS). It cannot be defined in a component file or Tailwind config — a Tailwind v4 project has no `tailwind.config.ts`.
- ⚠️ `pendingComponent` only fires when the loader is still pending. If data is already in the React Query cache, the loader resolves instantly and `pendingComponent` never mounts.

## Dependencies

None beyond existing TanStack Router.
