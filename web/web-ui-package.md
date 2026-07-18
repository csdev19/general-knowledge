# Web UI Package

_Complete guide to a shared `web-ui` package: theming, colors, and adding shadcn components._

The `@repo/web-ui` package is a shared UI component library that provides reusable React components, styles, and utilities for all applications in the monorepo.

## Overview

The web-ui package centralizes:

- **UI Components**: shadcn/ui components (Button, Card, Table, etc.)
- **Styles**: Tailwind CSS configuration and theme variables
- **Utilities**: Shared utility functions (e.g., `cn` for className merging)

## Installation

The package is part of the monorepo workspace. No installation needed - it's automatically available to all apps.

### Using in Your App

1. **Import the styles** in your app's main CSS file:

```css
/* apps/web/src/index.css */
@import "@repo/web-ui/styles.css";
```

2. **Import components** in your React components:

```tsx
import { Button, Card, CardContent, CardHeader, CardTitle } from "@repo/web-ui";
```

## Package Structure

```
packages/web-ui/
├── src/
│   ├── components/          # React components
│   │   ├── button.tsx
│   │   ├── card.tsx
│   │   └── ...
│   ├── lib/
│   │   └── utils.ts         # Utility functions
│   ├── styles.css           # Tailwind CSS and theme variables
│   └── index.ts             # Barrel exports
├── components.json          # shadcn/ui configuration
├── tailwind.config.ts       # Tailwind configuration
└── vite.config.ts           # Vite build configuration
```

## Adding shadcn Components

The package uses shadcn/ui for component management. All components are managed in the `web-ui` package.

### Adding a New Component

1. **Navigate to the web-ui package**:

```bash
cd packages/web-ui
```

2. **Add the component using shadcn CLI**:

```bash
npx shadcn@latest add dialog
```

Or from the root directory:

```bash
npx shadcn@latest add dialog --cwd packages/web-ui
```

3. **Export the component** from `src/index.ts`:

```tsx
// packages/web-ui/src/index.ts
export * from "./components/dialog";
```

4. **Use in your app**:

```tsx
import { Dialog, DialogContent, DialogTrigger } from "@repo/web-ui";
```

### Example Components

A mature package typically exports something like:

- Accordion
- Alert
- AlertDialog
- Avatar (+ AvatarImage, AvatarFallback, AvatarBadge, AvatarGroup, AvatarGroupCount)
- Badge
- Button
- Card
- Checkbox
- Dialog
- DropdownMenu
- Input
- Label
- Select
- Skeleton
- Sonner (Toaster)
- Table
- Textarea

## Theming and Colors

### Theme Variables

The theme is defined using CSS variables in `packages/web-ui/src/styles.css`. With Tailwind v4, tokens live in a `@theme inline` block; Tailwind auto-generates `bg-*`, `text-*`, `border-*` utilities from every `--color-{name}` variable — no `tailwind.config.js` needed.

Key facts for a Tailwind v4 setup:

- All tokens are defined in `packages/web-ui/src/styles.css` inside a `@theme inline` block using the **OKLCH** color format.
- Keep a brand accent token independent from your semantic tokens (e.g. keep the brand accent separate from green = positive and red = negative).
- Keep compatibility aliases for legacy shadcn names (`--color-background`, `--color-primary`, etc.) to avoid breaking third-party code.

### Adding Custom Colors

To add a custom color that can be used with Tailwind classes (e.g., `bg-example`, `text-example`):

1. **Add the color variable** in `packages/web-ui/src/styles.css`:

```css
:root {
  /* ... existing variables ... */
  --example: rgb(255, 0, 0); /* Your color value */
}

.dark {
  /* ... existing variables ... */
  --example: rgb(255, 0, 0); /* Dark mode color (can be different) */
}
```

2. **Register it in the `@theme inline` block**:

```css
@theme inline {
  /* ... existing theme variables ... */
  --color-example: var(--example);
}
```

3. **Use in your components**:

```tsx
<div className="bg-example text-white">Example colored div</div>
```

With Tailwind v4 you can also define the token directly in the `@theme inline` block, always using OKLCH:

```css
@theme inline {
  --color-example: oklch(0.5 0.2 250);
}
```

Then use as `bg-example`, `text-example`, `border-example`.

### Available Theme Colors

A typical package includes these theme colors:

**Base Colors:**

- `background` / `foreground`
- `card` / `card-foreground`
- `popover` / `popover-foreground`
- `primary` / `primary-foreground`
- `secondary` / `secondary-foreground`
- `muted` / `muted-foreground`
- `accent` / `accent-foreground`
- `destructive`
- `border`
- `input`
- `ring`

**Chart Colors:**

- `chart-1` through `chart-5`

**Sidebar Colors:**

- `sidebar` / `sidebar-foreground`
- `sidebar-primary` / `sidebar-primary-foreground`
- `sidebar-accent` / `sidebar-accent-foreground`
- `sidebar-border` / `sidebar-ring`

### Modifying Theme Colors

To change existing theme colors:

1. **Edit the CSS variables** in `packages/web-ui/src/styles.css`:

```css
:root {
  --primary: oklch(0.5 0.2 250); /* Change primary color */
  --destructive: oklch(0.6 0.25 20); /* Change destructive color */
}
```

2. **Rebuild the package** (if needed):

```bash
cd packages/web-ui
bun run build
```

3. **Changes apply immediately** in development mode (no rebuild needed for CSS changes)

### Color Format

The package uses **OKLCH** color format for better color consistency and easier manipulation:

```css
--primary: oklch(lightness chroma hue);
```

- **Lightness**: 0-1 (0 = black, 1 = white)
- **Chroma**: 0-0.4 (saturation)
- **Hue**: 0-360 (color wheel position)

Example:

- `oklch(0.5 0.2 250)` = Medium lightness, moderate saturation, blue hue

## Development Workflow

### Building the Package

```bash
cd packages/web-ui
bun run build
```

### Watch Mode

For development with automatic rebuilding:

```bash
cd packages/web-ui
bun run dev
```

### Type Checking

```bash
cd packages/web-ui
bun run check-types
```

## Best Practices

### 1. Always Import Styles

Make sure your app imports the styles from the package:

```css
@import "@repo/web-ui/styles.css";
```

### 2. Use Barrel Exports

Import components from the main package export:

```tsx
// ✅ Good
import { Button, Card } from "@repo/web-ui";

// ❌ Bad
import { Button } from "@repo/web-ui/components/button";
```

### 3. Add Components to Exports

When adding new components, always export them from `src/index.ts`:

```tsx
// packages/web-ui/src/index.ts
export * from "./components/your-new-component";
```

### 4. Keep Styles Centralized

All theme-related styles should be in `packages/web-ui/src/styles.css`. Don't duplicate theme variables in app-specific CSS files.

### 5. Use Theme Colors

Prefer theme colors over hardcoded colors:

```tsx
// ✅ Good
<div className="bg-primary text-primary-foreground">Content</div>

// ❌ Bad
<div className="bg-blue-500 text-white">Content</div>
```

## Troubleshooting

### Components Not Styled

If components appear unstyled:

1. Verify styles are imported: `@import "@repo/web-ui/styles.css";`
2. Check that Tailwind is processing the package's CSS
3. Ensure the package is built: `bun run build` in `packages/web-ui`

### Colors Not Working

If custom colors don't work:

1. Verify the variable is in both `:root` and `.dark`
2. Check it's registered in the `@theme inline` block
3. Rebuild the package if needed
4. Clear browser cache

### shadcn Components Not Found

If shadcn CLI can't find components:

1. Ensure you're in `packages/web-ui` directory or using `--cwd` flag
2. Check `components.json` exists and is properly configured
3. Verify path aliases in `tsconfig.json` match `components.json`

## Configuration Files

### components.json

Located at `packages/web-ui/components.json`. This file configures shadcn/ui:

```json
{
  "style": "base-lyra",
  "rsc": false,
  "tsx": true,
  "tailwind": {
    "config": "tailwind.config.ts",
    "css": "src/styles.css",
    "baseColor": "neutral",
    "cssVariables": true
  },
  "aliases": {
    "components": "@/components",
    "utils": "@/lib/utils"
  }
}
```

### tailwind.config.ts

Tailwind configuration for the package. Often minimal — content scanning is handled by consuming apps. With Tailwind v4 you may have no config file at all.

## Migration Notes

If you're migrating from app-specific components:

1. **Remove local UI components** from `apps/web/src/components/ui/`
2. **Update imports** to use `@repo/web-ui`
3. **Import styles** from the package instead of local CSS
4. **Remove duplicate theme variables** from app CSS files

> Building the package before a production build matters: the package exports point to built `dist/` files. If you get "Cannot find module" errors, run `bun run build` inside `packages/web-ui/`. See [Local Preview](./local-preview.md) for why dev and prod resolve workspace packages differently.

> **One Tailwind build per app — the app owns it.** The package must **not** inject its
> own Tailwind utilities layer (e.g. an `import "./styles.css"` in the package's
> `src/index.ts` combined with `vite-plugin-lib-inject-css`). The app already
> `@import`s the package's `styles.css` (source) and its `@source` scans the package's
> components, so the app's build generates every class the components use. A second
> utilities layer shipped from the package `dist` loads as a separate stylesheet in prod
> and silently overrides app-only responsive variants (`md:`/`lg:`) — the
> [tailwind-v4-split-css-cascade](./tailwind-v4-split-css-cascade.md) bug.
