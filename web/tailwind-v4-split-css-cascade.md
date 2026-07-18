# Tailwind v4 — Duplicate Utilities Across Split Stylesheets (prod-only responsive break)

_A responsive layout that works in `bun dev` but breaks in the production build:
on desktop, the app renders its **mobile** layout (single column, a "Show Map" /
overlay toggle) even though the viewport is wide. The `md:`/`lg:` display toggles
silently stop applying in prod. Root cause: a **duplicated Tailwind utilities layer**
across code-split stylesheets, where base utilities in the last-loaded sheet override
app-only responsive variants in the first._

> **Search keywords:** prod styles different from dev · responsive not working in
> production · `md:hidden` / `md:flex` / `lg:` ignored in prod · desktop shows mobile
> layout · two-column layout collapses to one column in prod · Tailwind v4 duplicate
> `@layer utilities` · shared UI package injects its own Tailwind · `libInjectCss`
> duplicate CSS · TanStack Start split CSS `index-*.css` vs `main-*.css` cascade.

## Symptom

- **Dev (`vite dev`)**: layout correct — e.g. two columns (sidebar | map).
- **Prod build (`vite build` + `vite preview`, or deployed)**: the **base** layout
  wins — the element with `hidden md:flex` stays `display:none`, the element with
  `flex md:hidden` stays `display:flex`, on a desktop-width window.
- Confirmed at runtime: `matchMedia('(min-width:48rem)').matches === true`, the
  `md:*` classes **exist** in the CSS, yet `getComputedStyle(el).display` is the
  base value, not the responsive one.

The tell: it only reproduces in a **production-equivalent build**, never in dev. Use
[local-preview](./local-preview.md) (`preview:local`) to reproduce it locally.

## Root cause

The production build **code-splits CSS** into more than one stylesheet, loaded in a
fixed order (e.g. TanStack Start emits `index-*.css` then `main-*.css`). Tailwind v4
puts utilities in `@layer utilities`. Within that layer, **source order decides** ties
(equal specificity; `@media` does not change specificity).

The problem is a **duplicated Tailwind utilities layer**: the same base utilities
(`.flex`, `.hidden`, …) are emitted into **two** stylesheets. The app-only responsive
variants (`.md\:flex`, `.md\:hidden`) are emitted into **only the first** sheet
(because only the app — not the shared UI package — uses them). So the effective
cascade across both files is:

```
index-*.css  @layer utilities { .flex{display:flex} … @media(min-width:48rem){ .md\:hidden{display:none} } }
main-*.css   @layer utilities { .flex{display:flex} … }     ← loads LAST, no .md\:hidden
```

Both `.flex` (base) and `.md\:hidden` (responsive) are in the **same layer**. The
**last** `.flex` (from `main-*.css`) comes after the first sheet's `.md\:hidden`, so it
wins → the element stays `display:flex` at desktop width → mobile layout on desktop.

### Why dev doesn't show it

Dev serves a **single** unified stylesheet with Tailwind's canonical order (base before
responsive within the layer), so `.md\:hidden` correctly comes last and wins. The bug
is created by the **split + duplication**, which only happens in the prod build.

### Why it only bites *app-only* responsive classes

If the responsive class were also used inside the shared UI package, it would land in
`main-*.css` too and win there. The break happens specifically for `md:`/`lg:` variants
used **only in the app** (e.g. a `TripSidebarLayout` with `hidden md:flex` /
`flex md:hidden`) that never appear in the package's scanned output.

## Where the duplication comes from (two sources — fix both)

The app must have a **single source of Tailwind**. Two things commonly create a second:

1. **The app imports its global CSS twice.** A side-effect `import "./index.css"` (e.g.
   in `router.tsx`) **bundles** the CSS into a JS chunk *in addition to* the canonical
   `<link>` (`import appCss from "./index.css?url"` in the root route). Keep only the
   `?url` + `links` version; delete the side-effect import.

2. **The shared UI package injects its own Tailwind build.** A `import "./styles.css"`
   in the package's `src/index.ts` (with `@tailwindcss/vite` + `vite-plugin-lib-inject-css`)
   ships a full Tailwind utilities layer inside the package `dist`, which gets injected
   into the app's chunk (`main-*.css`). Remove it — the **app** already owns Tailwind:
   its `index.css` does `@import "@scope/web-ui/styles.css"` (source), and that file's
   `@source "./components/**/*.tsx"` makes the app's build scan the package's components
   and generate every class they use. The package should ship **components**, not a
   second Tailwind build.

## The rule

**One Tailwind build per app, owned by the app.** The shared UI package contributes its
theme/tokens and component source (scanned via `@source`), but never emits or injects its
own utilities layer. Anything that produces a second `@layer utilities` in a separately
loaded stylesheet is a latent cascade bug.

## How to detect / verify

Run a production-equivalent build ([local-preview](./local-preview.md)) and grep the
emitted CSS — a base utility should appear **once**:

```bash
for f in apps/web/dist/client/assets/*.css; do
  echo "$f: .flex x$(grep -oF '.flex{' "$f" | wc -l)"
done
# BAD:  two files each contain .flex{  (duplicated utilities layer)
# GOOD: one file has the full utilities incl. .md\:hidden{; the other has ~none
```

At runtime, if `matchMedia('(min-width:48rem)').matches` is `true` but a
`hidden md:flex` element's computed `display` is `none`, you have this bug.

## Related

- [local-preview](./local-preview.md) — how to reproduce prod locally (this bug is
  invisible in dev).
- [web-ui-package](./web-ui-package.md) — the shared UI package build (the `development`
  → src / `import` → dist conditions, `@source` scanning).
- [bundle-splitting](./bundle-splitting.md) — how prod splits JS/CSS in the first place.
