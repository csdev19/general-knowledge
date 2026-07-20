# Tagging system (two-tier: default + custom)

_The standard way to tag domain entities (locations, shopping items, …) across
NIWAY apps. Proven in trip-planner and shopping-list; reuse it, don't reinvent._

A tag system that stays consistent, needs zero per-scope seeding, and shares one
source of truth between web, mobile and native widgets.

## The two tiers

**1. Default tags — defined in CODE, not in the DB.**
The fixed, everyone-gets-them classifications (grocery aisles, place categories…).
They live as a constant array in the domain package, so there are **zero DB rows**
to seed per list/trip and no way for them to drift between clients.

```ts
export interface DefaultTag {
  slug: string;   // stable id, stored on the entity (e.g. "produce")
  label: string;  // display name ("Produce")
  emoji: string;  // "🥬"
  color: string;  // hex, from the palette below
}

export const DEFAULT_TAGS: DefaultTag[] = [ /* … */ ];
export const DEFAULT_TAG_SLUGS = DEFAULT_TAGS.map((t) => t.slug);

export function getDefaultTag(slug?: string): DefaultTag | undefined {
  return slug ? DEFAULT_TAGS.find((t) => t.slug === slug) : undefined;
}
/** First default tag = the entity's "primary" (drives its emoji/color). */
export function getPrimaryDefaultTag(slugs?: string[]): DefaultTag | undefined {
  return getDefaultTag(slugs?.[0]);
}
```

**2. Custom tags — per-scope DB rows.**
User-created tags scoped to a list/trip. Stored in a `tags` table and referenced
by id. The tag carries a **color key** (not a raw hex) from a fixed palette, so
the set of colors stays curated and consistent.

```ts
// schema: tags = { scopeId, name, color /* palette key */, createdBy, ... }
export const TAG_COLORS = {
  coral: { label: "Coral", hex: "#FF6B6B" },
  violet: { label: "Violet", hex: "#A29BFE" },
  amber: { label: "Amber", hex: "#FDCB6E" },
  sky: { label: "Sky", hex: "#74B9FF" },
  emerald: { label: "Emerald", hex: "#00B894" },
  rose: { label: "Rose", hex: "#FD79A8" },
  indigo: { label: "Indigo", hex: "#6C5CE7" },
  lime: { label: "Lime", hex: "#A3CB38" },
  orange: { label: "Orange", hex: "#E17055" },
  cyan: { label: "Cyan", hex: "#00CEC9" },
} as const;
export type TagColorKey = keyof typeof TAG_COLORS;

export function getTagColorHex(colorKey: string): string {
  return TAG_COLORS[colorKey as TagColorKey]?.hex ?? TAG_COLORS.coral.hex;
}
```

## How an entity references tags

An entity stores **both** references — defaults by slug, custom by id:

```ts
{
  defaultTags?: string[];      // slugs into DEFAULT_TAGS  (e.g. ["produce"])
  tagIds?: Id<"tags">[];       // ids into the per-scope tags table
}
```

Rendering (a card/row) resolves both into chips:

```ts
const defaultChips = (entity.defaultTags ?? []).map(getDefaultTag).filter(Boolean);
const customChips  = tags.filter((t) => entity.tagIds?.includes(t._id));
// emoji/primary color for the whole entity = the first default tag:
const emoji = getPrimaryDefaultTag(entity.defaultTags)?.emoji ?? FALLBACK_EMOJI;
```

## Why this shape

- **No seeding, no drift.** Defaults are code, so every scope has the same set
  instantly and web/mobile can never disagree. Adding a default = one array entry.
- **Curated colors.** Custom tags pick a palette **key**, not an arbitrary hex, so
  the UI stays coherent (and dark-mode tints are derived, e.g. `hex + "1a"`).
- **One source of truth.** All of the above lives in the **domain package**
  (`@app/domain/constants`), imported by Convex functions, web and mobile alike —
  the same rule as [constants-pattern.md](./constants-pattern.md) and
  [schemas-first.md](./schemas-first.md).
- **Stable slugs.** Store the slug, not the label — labels/emojis can change
  without a data migration.

## Migration note (category → defaultTags)

Older schemas may have a single `category: string`. Treat it as a transitional
alias: `defaultTags ?? (category ? [category] : [])`. New writes use `defaultTags`;
drop `category` after backfilling.

## Reference implementations

- **trip-planner** — `@trip-planner/domain/constants` (`DEFAULT_TAGS` from place
  categories, `TAG_COLORS`, `getTagColorHex`, `getDefaultTag`,
  `getPrimaryDefaultTag`); rendered in the location card + map pins.
- **shopping-list** — `@shoping-app/domain/constants` (default tags = grocery
  aisles); rendered on item rows.
