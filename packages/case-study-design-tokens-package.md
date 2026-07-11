# Case study: building a shared design-tokens package end to end

_A worked example of creating a new shared workspace package: one typed TS source of truth generating the CSS custom properties consumed by a desktop app and a web app, with dark + light themes, shipped as three sequential PRs._

> **Status: worked example.** This case study generalizes a real design so you can reuse the
> pattern for any new shared package. It replaces a manual token mirror between two apps (a desktop
> app's `base.css` as the source of truth today and a hand-copied subset in the web app) with a
> single generated package.

## Problem

The brand palette lives in two places that drift silently: a desktop app's `base.css` `:root` block
(~60 tokens: colors, spacing, radii, typography, transitions, layout dims) and a hand-mirrored
10-token subset in the web app, prefixed `--app-*` to avoid clashing with the UI library's variables
(`--border`, `--primary`, …). There is no light theme anywhere, and JS code that needs raw values
(canvas rendering, export pipelines) hardcodes hex strings.

## Goal

One typed TS source of truth that generates every consumed form of the tokens:

- CSS custom properties for the desktop (unprefixed — existing names, zero churn).
- CSS custom properties for the web (`--app-*` prefixed — zero collision with the UI library).
- Importable TS objects for JS consumers (canvas, exports).
- Dark **and** light themes, with a toggle on the web app and a theme setting in the desktop app.
  Dark remains the default everywhere.

## Non-goals

- Migrating the UI library's design tokens (`--background`, `--primary`, …) into this package. Those
  routes keep the library's system untouched.
- A `system` (auto) theme option — follow-up; both surfaces ship `dark | light`.
- Replacing hardcoded hexes in canvas/export code — the TS export makes it _possible_; the migration
  is opportunistic follow-up work, not part of these PRs.
- Style Dictionary / W3C token JSON tooling — ~60 tokens and 2 targets don't justify the dependency.
  Revisit if a third platform (e.g. mobile) appears.

## Architecture

### Package layout

```
packages/tokens/
  src/
    types.ts          # TokenSet — the typed shape every theme must satisfy
    base.ts           # theme-invariant tokens: spacing, radii, typography,
                      # transitions, layout dims (--sidebar-width, --titlebar-height)
    themes/
      dark.ts         # colors + shadows/glows — today's base.css values, verbatim
      light.ts        # same shape, light values (designed in PR2)
    index.ts          # exports { base, dark, light } for JS/TS consumers
    generate.ts       # TS → CSS generator (~80 lines, zero deps)
  css/
    tokens.css        # GENERATED + COMMITTED — unprefixed (desktop)
    tokens.app.css    # GENERATED + COMMITTED — --app-* prefixed (web)
  package.json        # tsdown + devExports pattern (see shared-package-build-strategy)
  tsdown.config.ts
```

### Key decisions

1. **`base` vs `themes` split.** Spacing/radii/fonts don't vary by theme, so they live in `base.ts`
   and are emitted once under `:root`. Theme files carry only colors and shadow/glow values; the
   generated `[data-theme="light"]` block re-declares only those.
2. **Type-enforced theme parity.** `dark` and `light` are both `TokenSet`. A theme missing a token
   is a compile error — an incomplete theme cannot ship.
3. **Generated CSS is committed, guarded by a drift test.** A vitest test runs the generator in
   memory and diffs against `css/*.css`; editing a TS token without running `bun run generate` fails
   CI with a clear message. Same committed-artifact pattern as a built `dist/`, plus the guard it
   lacks.
4. **Two CSS flavors, one source.** The generator emits unprefixed names for the desktop (every
   existing `var(--border)` in CSS modules keeps working untouched) and `--app-*` prefixed names for
   the web (the UI library's `--border`/`--primary` are never shadowed). The full token set is
   emitted in both flavors — no curation.
5. **CSS theme structure.** `tokens.css` = `:root { base + dark }` + `[data-theme="light"] { light
   colors }`. No attribute → dark, exactly today's rendering. Light is strictly additive.
6. **Exports.** `.` → TS objects (tsdown-built, `devExports` keeps dev pointed at `src/`); `./css` →
   `css/tokens.css`; `./css/app` → `css/tokens.app.css`. The CSS files are static exports — tsdown
   doesn't touch them. The bundler resolves bare-specifier `@import` via the exports map in both apps
   (precedent: `@import "@scope/web-ui/styles.css"`).
7. **Formatting.** The generator's output must be formatter-stable (generate, then CI's
   formatter `--check` sees no diff). If the formatter fights the generated layout, exclude the
   generated CSS dir the way generated `CHANGELOG.md` is excluded.

### Consumption

- **Desktop:** `base.css` replaces its hand-written `:root` token block with
  `@import "@scope/tokens/css";`. Everything else in `base.css` (scrollbar theming, body defaults,
  resets) stays. No CSS-module changes — token names are identical.
- **Web:** the global stylesheet replaces the `--app-*` mirror block with
  `@import "@scope/tokens/css/app";`. The few references to the mirror's divergent names
  (`--app-accent` → `--app-accent-primary`, `--app-accent-hover` → `--app-accent-primary-hover`) are
  renamed to match the generated names.
- **JS consumers (later):** `import { dark } from "@scope/tokens"` wherever a raw hex is needed at
  runtime.

### Theming model

- **Web (PR2):** a toggle sets `data-theme` on the **app wrapper div** (not `<html>` — a global
  attribute leaks the app vars into routes that use the UI library's own tokens) and persists to
  `localStorage` (`app-theme`); default is dark. Accepted tradeoff: returning light-theme visitors
  briefly see dark until hydration (cookie-based SSR theme is the upgrade path if it ever matters).
- **Desktop (PR3):** a settings value `AppSettings.theme` (typed, validated, persisted, broadcast)
  is narrowed to `"light" | "dark"` (default `"dark"`; any legacy persisted `"system"` coerces to
  dark on merge) and surfaced: the shared renderer root (the single mount point for all windows)
  applies `document.documentElement.dataset.theme` on load and on every `settings:changed`
  broadcast. A theme selector ships on the Settings page.

### Light palette constraints (values designed in PR2, not here)

- The brand accent color stays the accent in both themes.
- Every text-on-background token pair meets WCAG AA contrast.
- Shadows/glows get light-specific values — the dark ones are calibrated for near-black backgrounds
  and will look muddy on white.
- Transparent/floating windows are a known trap (background + shadow interplay) — explicit QA items.
- Semantic accents (green/red/yellow/purple + the brand accent) meet WCAG 1.4.11 non-text contrast
  (3:1) on each distinct light surface (`bg-app`, `bg-card`, `bg-sidebar`) — guarded in a
  `contrast.test.ts`. (Some semantic colors were darkened in light to clear the bar.)

## Delivery — three sequential PRs, owner-validated each

| PR  | Scope                                                                                                                     | Validation                                                                                 |
| --- | ------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| PR1 | `@scope/tokens` package (dark real, light placeholder = dark values); desktop + web consume it; web rename; drift test    | Pure refactor: both apps render **byte-identical**. Visual spot-check + full CI            |
| PR2 | Design `light.ts` for real; web theme toggle                                                                              | Owner reviews light web app in browser; contrast checks                                    |
| PR3 | Desktop `theme` setting (settings + broadcast + Settings UI); light QA across all windows                                 | Owner walks every window/editor in light in a packaged build; transparent-window checklist |

Each PR is independently valuable; a stall in PR3's QA doesn't block the already-shipped package or
the web light mode.

## Testing

- **Drift test** (PR1): generator output === committed CSS.
- **Generator unit tests** (PR1): token → CSS emission, prefixing, theme block structure.
- **Type parity** (PR1): compile-time via `TokenSet` — no runtime test needed.
- **Existing suites stay green** (all PRs): desktop unit tests, E2E harness, both app builds. PR1's
  bar is "no visual or behavioral change".
- **PR3 manual QA checklist:** every window in light theme, transparent-window rendering, and a
  mid-session theme switch reaching every open window.

## Risks

- **Committed generated files invite hand-edits.** Mitigated by the drift test and a header comment
  in each generated file ("GENERATED — edit src/themes/\*, run `bun run generate`").
- **Desktop light QA is the long tail.** Isolated to PR3 by design; PR1/PR2 value ships regardless.
- **Formatter vs generated CSS.** Resolved either by emitting formatter-stable output or excluding
  the generated dir (both precedented); decided during PR1.

## Reusable takeaways

- A shared package can ship **generated, committed artifacts** (CSS here) as long as a **drift test**
  guards them against hand-edits — the same discipline applies to any generated output.
- **Type-enforce parity** between variant sets (themes here) so an incomplete variant is a compile
  error, not a runtime bug.
- **One source, many flavors:** a small generator (zero deps) can emit multiple consumption targets
  (unprefixed vs prefixed CSS, TS objects) from a single typed source.
- **Static assets as package exports:** non-code files (CSS/JSON) can be exposed via the `exports`
  map alongside tsdown-built entry points; the bundler resolves bare-specifier `@import`s through it.
- **Sequence delivery by independent value:** structure the rollout so each PR ships value even if a
  later, riskier PR stalls.
