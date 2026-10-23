# Electron — app icon, menu-bar icon, dev Dock icon and a branded DMG window

_The Electron / electron-builder recipe for the first impression on macOS: a correctly sized app
icon, a menu-bar icon that follows light and dark mode, the right Dock icon in development, and an
install window that is not electron-builder's default grey one._

**In one minute.** Keep SVG sources in `build/` and render everything from them. Draw the app icon
on Apple's grid (824 px tile inside a 1024 canvas). Make the tray icon a black-and-alpha template
image and call `setTemplateImage(true)`. In development, set the Dock icon yourself, because the
stock Electron binary has its own. For the DMG, add a `dmg:` block with a **light** background PNG
(plus an `@2x`), `iconSize`, `window` and two `contents` entries whose coordinates match the arrow
drawn in the background. Check it by opening a real unsigned DMG.

The geometry and the Finder-label trap are explained in
[macos-app-icon-and-dmg.md](./macos-app-icon-and-dmg.md), which captures the layout in a
`.DS_Store` and builds the image with `hdiutil`. electron-builder generates that layout from the
`dmg:` keys instead, so this page covers only what changes in Electron. The signing and update
side is [electron-macos.md](./electron-macos.md).

**Status.** Icon, favicon, tray template and the DMG window were checked on a real build (the DMG
by building and opening an unsigned image). The dev Dock icon and the Windows icon derivation were
**not** verified; they are marked below.

---

## Checklist for a new app

- [ ] `build/icon-source.svg` on the 1024 canvas, 824 px tile, ~185 px radius, 100 px margin
- [ ] `build/icon.png` (1024, with alpha), `build/icon.icns`, and `resources/icon.png` (512) rendered from it
- [ ] `build/tray-source.svg`: black glyph, alpha only, details cut out; `tray.png` (18) and `tray@2x.png` (36)
- [ ] Tray code calls `setTemplateImage(true)` on `darwin`
- [ ] Main sets `app.dock.setIcon(...)` when `!app.isPackaged`, at startup and after each `app.dock.show()`
- [ ] `build/dmg-background.svg` (light, 660×400) rendered to `dmg-background.png` and `dmg-background@2x.png` (1320×800)
- [ ] `dmg:` block in `electron-builder.yml` with coordinates matching the SVG
- [ ] Product site favicon from the same art, full-bleed (see below)
- [ ] Built and opened an unsigned DMG and looked at the window
- [ ] Shipped in the same release as the icon change (the DMG only changes with the next release)

---

## App icon

Follow the grid from [macos-app-icon-and-dmg.md](./macos-app-icon-and-dmg.md): 1024 canvas, 824 px
rounded tile (rx about 185), 100 px transparent margin. Two failures come from skipping it. A
full-bleed 1024 tile looks oversized next to other Dock icons. And a PNG without alpha shows white
corners in Finder and Quick Look.

electron-builder takes a single source and does not add the margin, so the margin must already be
in the source. Render from the SVG, for example with `sharp` (librsvg), using a density
proportional to the target size so small sizes stay sharp:

```text
build/icon.png            1024      (alpha)
build/icon.icns           from an .iconset: 16, 32, 128, 256, 512 at 1x and @2x
resources/icon.png        512       (used for the window/Dock icon at runtime)
```

Build the `.icns` with `iconutil -c icns <name>.iconset`. Deleting a committed `icon.ico` lets
electron-builder derive the Windows icon from `icon.png`; this is what its docs describe, and it
was **not verified on a Windows build**.

### Favicon for the product site

Use the same art **full-bleed**, with no grid margin: browser tabs have no Dock to match. Pack an
`.ico` with embedded PNG entries at 16, 32, 48 and 256, and add a 180 px `apple-touch-icon.png`
linked from `<head>`.

---

## Menu-bar (tray) icon

On macOS the menu bar tints a **template image**. Provide a black glyph on a transparent
background, with inner details (eyes, counters) cut out so they show through, at 18 px and 36 px
(`@2x` suffix, so Electron picks it for Retina):

```ts
const image = nativeImage.createFromPath(join(__dirname, "../../resources/tray.png"));
if (process.platform === "darwin") image.setTemplateImage(true);
const tray = new Tray(image);
```

Only the alpha channel is used. A coloured icon stays coloured on both bars and is unreadable on
one of them; with no custom image at all you get Electron's default atom.

## Dock icon in development

`bun run dev` (or any `electron .`) runs the stock Electron binary, so macOS shows Electron's icon in
the Dock and Cmd+Tab. The bundle's `.icns` only exists in packaged builds. Set it from main:

```ts
function applyDevDockIcon(): void {
  if (process.platform === "darwin" && !app.isPackaged) app.dock?.setIcon(iconPath);
}
// call at startup, and again after every dock.show():
void app.dock?.show().then(applyDevDockIcon);
```

Re-apply after `dock.show()` because showing the Dock can bring back the bundle's icon. **Not
verified live** when this was written: it follows Electron's API, but nobody watched the Dock.

---

## Branded DMG window

```yaml
# electron-builder.yml
dmg:
  artifactName: ${name}-${version}-${arch}.${ext}   # ${arch} avoids arm64/x64 name collisions
  background: dmg-background.png                    # relative to buildResources (build/)
  iconSize: 128
  window:
    width: 660
    height: 400
  contents:                                         # icon CENTRES, in window points
    - x: 180
      y: 210
    - x: 480
      y: 210
      type: link
      path: /Applications
```

### Assets

| File | Size | Notes |
| --- | --- | --- |
| `build/dmg-background.svg` | 660×400 | source of truth; comment states the coordinates it assumes |
| `build/dmg-background.png` | 660×400 | `dmg.background` |
| `build/dmg-background@2x.png` | 1320×800 | same directory, same name plus `@2x` |

dmg-builder pairs the `@2x` file with the base name for Retina: in `dmg-builder@26.18.0`
(`out/dmgUtil.js`) it derives `<name>@2x.<ext>` from the background path and, if present, merges
both with `tiffutil -cathidpicheck`. Render both PNGs from the same SVG.

### Design rules, with the reason

- **Light background.** Finder draws icon labels in dark text and electron-builder exposes no
  label-colour option. A dark background makes "Applications" hard to read. (Same trap as in
  [macos-app-icon-and-dmg.md](./macos-app-icon-and-dmg.md).)
- **Brand mark and name at the top, then one short instruction line.** The window has one job:
  say "drag this into Applications".
- **Draw the arrow in the background, between the two icon centres.** Finder places icons by
  `contents`; the background cannot move them. If you change `x`/`y` in the YAML, change the SVG
  too. The comment at the top of the SVG is the guard against drift.
- **Keep important content in the top ~330 px.** Finder's title bar and toolbar take space from
  the 400 px window. A footer line at y about 366 was cut off in the reference build.
- **Keep the SVG in `build/`**, not only the PNGs, so the next art revision is one render away.

---

## Verify without signing

Stage whatever `extraResources` the build requires (bundled engines, models), then:

```bash
bun run build
CSC_IDENTITY_AUTO_DISCOVERY=false electron-builder --mac dmg --arm64 --publish never
open dist/*.dmg
```

Look at the window: background, arrow between the icons, label legibility, nothing cut off. This
proves the layout only; signing, notarization and `spctl` are covered by the spine in
[README.md](./README.md).

---

## Applied in

Reference app `invisible-assistant` (`apps/desktop`):

- Icon, tray, Dev Dock: PR #265 (`build/icon-source.svg`, `build/tray-source.svg`,
  `src/main/tray.ts`, `applyDevDockIcon` in `src/main/index.ts`)
- Favicon and Apple touch icon: PR #266
- DMG window: invisible-assistant PR #268 (`electron-builder.yml` `dmg:` block,
  `build/dmg-background.svg`, `dmg-background.png`, `dmg-background@2x.png`)
- Baseline it improves on: a sibling Electron app that ships electron-builder's default window
  (grey background, dashed arrow)

## Related

- [macos-app-icon-and-dmg.md](./macos-app-icon-and-dmg.md): icon geometry and the `.DS_Store` approach
- [electron-macos.md](./electron-macos.md): signing, feed and updates
- [README.md](./README.md): the release spine
- [../desktop/main-process-architecture.md](../desktop/main-process-architecture.md): where main-process code like the tray lives
