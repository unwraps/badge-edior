# EPIC Badge Editor — Scope & Requirements

## Overview

A **simple, lightweight** single-page **3D badge/nameplate previewer** built with SvelteKit & Three.js. This is not a complex 3D editor — it's a focused tool for previewing a badge and making basic edits. Users can customize a two-plate badge (top text, bottom text with status indicator circle, color, fonts) and export the result as an STL file for 3D printing.

Includes a **VRChat badge base mode** where a pre-bundled badge base model loads alongside the editable badge, with the text positioned correctly on the base so everything fits together for VRChat avatar use.

---

## 1. Tech Stack

| Layer | Technology | Version / Notes |
|---|---|---|
| Framework | **SvelteKit** | @latest (Svelte 5 runes mode enforced) |
| UI / Styling | **Tailwind CSS v4 (I think)** @latest + **DaisyUI v5** @latest + **bits-ui** @latest | All radii forced to 0 (squared-off look) |
| 3D Engine | **Three.js** | @latest |
| Font Loading | Three.js `FontLoader` + `TextGeometry` | Uses `helvetiker_regular.typeface.json` (bundled static asset) |
| Export | Three.js `STLExporter` | Produces ASCII STL |
| Package Manager | **pnpm** | v9+ workspace |
| Build Tool | **Vite** | v8 |
| Language | **TypeScript** | v6 (strict mode) |
| Adapter | `@sveltejs/adapter-auto` | For deployment |

**Key config:**
- Svelte 5 runes mode forced via `svelte.config.js` `compilerOptions.runes`
- TypeScript strict mode enabled
- Tailwind + DaisyUI with `@tailwindcss/vite` plugin (not PostCSS config file)
- Two DaisyUI themes: `dark` (default, prefers-dark) and `light`

---

## 2. Project Structure

```
badge-preview2/
├── package.json
├── pnpm-workspace.yaml
├── svelte.config.js
├── tsconfig.json
├── vite.config.ts
├── README.md
├── static/
│   ├── robots.txt
│   └── fonts/
│       └── helvetiker_regular.typeface.json    # Default 3D font
├── static/
│   ├── robots.txt
│   ├── fonts/
│   │   └── helvetiker_regular.typeface.json    # Default 3D font
│   └── models/
│       └── vrchat-badge-base.stl               # Pre-made VRChat badge base (clip/bracket model)
└── src/
    ├── app.d.ts                                 # (empty) App namespace
    ├── app.html                                 # Root HTML shell
    ├── lib/
    │   ├── index.ts                             # Placeholder barrel export
    │   └── assets/
    │       └── favicon.svg                      # Svelte logo SVG
    └── routes/
        ├── layout.css                           # Global CSS, Tailwind imports, DaisyUI themes
        ├── +layout.svelte                       # Root layout (favicon, renders children)
        └── +page.svelte                         # ** Main single-page app **
```

---

## 3. Features (Functional Requirements)

### 3.1 3D Scene

- Full-screen Three.js scene rendered on a `<canvas>` element.
- **Lighting:** Ambient light (0.6 intensity) + Directional light (0.8 intensity).
- **Background color:** `#efefef`.
- **Camera:** Perspective (50° FOV), initial position `(0, 0, 150)`.
- **Controls:** `OrbitControls` with damping — user can rotate/zoom/pan the badge.
- **Responsive:** Scene resizes with the browser window.
- **Cleanup:** Renderer disposed on component unmount.

### 3.2 Badge Geometry

Two rectangular plates stacked vertically with a gap:

| Plate | Width | Height | Depth |
|---|---|---|---|
| Top plate | 104.5mm | 15.5mm | 2mm |
| Bottom plate | 104.5mm | 9.5mm | 2mm |
| Gap between | 3mm | | |

- Material for both plates: `MeshStandardMaterial` color `#444444` (dark gray).
- Positioned symmetrically along the Y-axis.

### 3.3 Top Text

- **Default value:** `"TOP TEXT"`
- **Font:** Loaded from `/fonts/helvetiker_regular.typeface.json` by default.
- **Font override:** User can upload a **custom `.json` typeface file** for the top plate.
- **Sizing:** Font size = 60% of top plate height.
- **Extrusion depth:** 2mm.
- **Positioning:** Horizontally centered on the plate. Vertical baseline is stabilized using a reference string `"My"` to prevent jumping when text changes.
- Material: `MeshStandardMaterial` color `#ffffff` (white).

### 3.4 Bottom Text & Status Circle

- **Default value:** `"BOTTOM TEXT"`
- **Font:** Same default font, independently overridable — user can upload a **custom `.json` typeface file** for the bottom plate.
- **Sizing:** Font size = 60% of bottom plate height.
- **Extrusion depth:** 2mm.
- **Status indicator circle:** A cylinder (circle) placed to the **left** of the bottom text.
  - Radius: 3.5mm, depth: 2mm.
  - Color: user-selectable via a color picker (default `#FF0000` red).
  - Gap between circle and text: 10mm.
  - Entire bottom row (circle + text) is **centered horizontally** as a group.
- **Material:** White text, colored circle.

### 3.5 Status Presets

Four quick-apply buttons that set both the circle color and bottom text at once:

| Preset Name | Text | Color |
|---|---|---|
| Do not disturb | Do not disturb | `#FF0000` (red) |
| Ask | Ask | `#FFA500` (orange) |
| Active | Active | `#00AA00` (green) |
| Join Me | Join Me | `#00CCFF` (cyan) |

Button text color auto-adjusts for readability (black on cyan, white on others).

### 3.6 UI Panel (Controls)

A floating **sidebar card** on the left side of the screen containing:

1. **Top Text** — text input field
2. **Top Font Upload** — file input (accepts `.json` only), with error alert on invalid files
3. **Divider**
4. **Bottom Text** — text input field
5. **Bottom Font Upload** — file input (accepts `.json` only), with error alert on invalid files
6. **Divider**
7. **Circle Color** — native HTML color picker
8. **Status Presets** — 4 buttons in a column
9. **Grid Toggle** — toggle button to show/hide a ground grid in the 3D scene
10. **VRChat Base Mode Toggle** — toggle switch to enter/exit VRChat badge base mode; when activated, the pre-bundled base model loads automatically
11. **Badge Position Offset Controls** — X/Y/Z fine-tune sliders to adjust the badge's position relative to the base (visible in VRChat mode)
12. **Export STL Button** — primary action button

Layout:
- Card width: `w-72` (mobile) → `w-96` / `w-md` (larger screens)
- Card has backdrop blur, shadow, and semi-transparent background.
- Scrollable if content overflows.
- Responsive padding/sizing via `sm:` breakpoints.
- **Touch-friendly** on tablet and Android: all interactive elements have large tap targets (minimum 44px), inputs are sized for mobile keyboard use, and the card is positioned to avoid overlap with browser chrome on mobile viewports.

### 3.7 STL Export

- Uses `STLExporter` to export only the **badge itself** (top plate + top text + bottom plate + bottom text + status circle) as ASCII STL.
- The VRChat badge base is **not** included in the export — it is a visual reference only.
- Triggers a browser download named `badge.stl`.
- No server-side processing — everything runs client-side.

### 3.8 VRChat Badge Base Mode

A secondary mode for VRChat avatar accessory creation:

- **Mode toggle:** A switch in the UI panel to enter/exit VRChat badge base mode.
- **Pre-bundled base model:** The project ships with a ready-made VRChat badge base STL at `static/models/vrchat-badge-base.obj`. This model (e.g., a clip-on badge holder, breastplate attachment, or bracket with a badge recess) is loaded automatically when the user activates VRChat mode — no file picking needed.
- **Same-scale alignment:** The base model and the editable badge share the same coordinate system and scale. The badge text plates render at real-world size relative to the base, so they match physically when 3D-printed.
- **Badge slot/placeholder:** The pre-bundled base model is designed with a known reference point (world origin) so the badge (top plate, bottom plate, text, circle) auto-aligns into the designated recess/slot.
- **Fine-tune offset controls:** X, Y, and Z sliders in the UI panel let the user nudge the badge position relative to the base for precise alignment.
- **Visual feedback:** The badge base is rendered with a distinct material (e.g., semi-transparent or a different color) so the user can clearly see how the text badge fits into the base.
- **Export always excludes the base:** The STL export only outputs the badge plates + text + circle — the VRChat base is a visual reference only and is never included in the downloaded file.

**Note:** The base OBJ file at `static/models/vrchat-badge-base.obj` must be created or sourced separately and placed in the project before VRChat mode will function.

### 3.9 3D Scene Grid & Ground Plane

- **Grid helper:** A Three.js `GridHelper` (or custom grid) visible in the 3D scene to give spatial reference.
  - Grid lies on the XZ plane (ground) beneath the badge.
  - Default color scheme: subtle gray subdivisions.
  - Can be toggled on/off via a button in the UI panel.
- **Ground plane (optional):** A semi-transparent ground plane or shadow-receiving plane for additional depth perception.
- **Axes helper (optional):** Small XYZ axes indicator in the corner for orientation reference when positioning a VRChat base.

---

## 4. User Interactions (Data Flow)

1. User types text → `oninput` handler calls `updateTopText()` / `updateBottomText()` → text geometry is rebuilt, old meshes disposed, new mesh added to scene.
2. User picks a color → `updateCircleColor()` updates the `MeshStandardMaterial.color` in-place (no rebuild).
3. User clicks a preset → `applyPreset()` sets both color and bottom text, triggers both updates.
4. User uploads a font → `FileReader` reads JSON → `FontLoader.parse()` → font object stored → text mesh rebuilt with new font. Errors shown inline.
5. User toggles grid → `toggleGrid()` flips visibility of the `GridHelper` object in the scene.
6. User toggles VRChat mode → `toggleVrchatMode()` loads the pre-bundled base OBJ from `static/models/vrchat-badge-base.obj`, shows the base + badge + offset controls.
7. User adjusts badge position on base → X/Y/Z offset sliders update the badge group's position relative to the base in real-time.
8. User clicks Export → `STLExporter.parse(badgeGroup)` (always only the badge — plates, text, circle; VRChat base is excluded) → Blob → download link clicked → file saved.

---

## 5. Non-Functional Requirements / Constraints

- **Client-side only:** No backend, no API routes, no server load functions.
- **Responsive (desktop + tablet + Android):** Supports desktop, tablet, and Android mobile via Tailwind breakpoints (`sm:`, `md:`, `lg:`).
  - Touch targets: All buttons, inputs, toggles, and file uploads must have **minimum 44×44px tap targets** for touch usability.
  - Input sizing: Text inputs and file inputs must be comfortably tall for mobile keyboard use (`min-h-[44px]` or DaisyUI `input-md` equivalent).
  - Card positioning: On narrow viewports, the sidebar card should not overflow off-screen; it may collapse to a bottom sheet, a hamburger drawer, or a narrower column.
  - Prevent viewport zoom on focus: `<meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1">` or CSS `touch-action: manipulation` to avoid accidental zoom on input focus.
- **Dark theme default:** DaisyUI `dark` theme with `prefersdark: true`.
- **Squared UI:** All DaisyUI radius settings (`--radius-box`, `--radius-field`, `--radius-selector`) are forced to `0rem`.
- **Fonts are typeface JSON:** Not `.ttf`/`.woff` — must be Three.js `FontLoader`-compatible `.json` files (e.g., from `https://threejs.org/examples/fonts/`).
- **Pre-bundled OBJ for VRChat base:** The app loads `static/models/vrchat-badge-base.obj` at runtime via `OBJLoader` when VRChat mode is activated.
- **Grid helper:** Implemented via Three.js `GridHelper`; toggleable at runtime without scene rebuild.

---

## 6. How to Reproduce (for the next coding agent)

### 6.1 Scaffold

```bash
# Using sv CLI v0.15
pnpm dlx sv@0.15.0 create --template minimal --types ts \
  --add prettier tailwindcss="plugins:none" --install pnpm badge-preview2
```

Then manually add:
- Dev deps: `@tailwindcss/vite`, `daisyui`, `prettier-plugin-tailwindcss`
- Runtime dep: `three`, `@types/three`

### 6.2 Critical Configuration

1. **`vite.config.ts`** — Must use `@tailwindcss/vite` plugin (NOT PostCSS-based Tailwind).
2. **`svelte.config.js`** — Must force runes mode (see existing config).
3. **`layout.css`** — Must `@import 'tailwindcss'` and `@plugin 'daisyui'` (Tailwind v4 CSS-first config). Define two daisyui themes (dark + light) with all radii = 0.
4. **Font file** — Download `helvetiker_regular.typeface.json` from Three.js examples and place in `static/fonts/`.

### 6.3 Key Dependencies (from `package.json`)

```json
{
  "devDependencies": {
    "@sveltejs/adapter-auto": "^7.0.1",
    "@sveltejs/kit": "^2.57.0",
    "@sveltejs/vite-plugin-svelte": "^7.0.0",
    "@tailwindcss/vite": "^4.3.3",
    "@types/three": "^0.183.1",
    "daisyui": "^5.7.4",
    "prettier": "^3.8.1",
    "prettier-plugin-svelte": "^3.5.1",
    "prettier-plugin-tailwindcss": "^0.7.2",
    "svelte": "^5.55.2",
    "svelte-check": "^4.4.6",
    "tailwindcss": "^4.3.3",
    "typescript": "^6.0.2",
    "vite": "^8.0.7"
  },
  "dependencies": {
    "three": "^0.183.2"
  }
}
```

---

## 7. File-by-File Implementation Guide

### `src/routes/+page.svelte` (~500+ lines)

**Script section:**
- Import `onMount` from `svelte`
- Import `THREE`, `OrbitControls`, `FontLoader`, `TextGeometry`, `STLExporter`, `OBJLoader`, `GridHelper` from `three` / `three/examples/jsm/`
- Reactive state with `$state()`: `topText`, `bottomText`, `circleColor`, `topFontError`, `bottomFontError`, `showGrid`, `vrchatMode`, `importedBaseGroup`
- Presets array (4 objects with `name`, `text`, `color`)
- Scene objects: `scene`, `camera`, `renderer`, `controls`, `badgeGroup`, `fontLoader`, `topFont`, `bottomFont`, mesh refs, `circleMaterial`, `gridHelper`, `vrchatBaseGroup`
- Plate dimensions constants
- `onMount`: initialize scene, load default font
- `initScene()`: create scene, camera, renderer, controls, lights, plates, grid helper (initially hidden), start animation loop, resize listener
- `updateTopText()`: dispose old mesh, build new `TextGeometry`, center horizontally, stabilize vertical baseline with `"My"` ref geometry
- `updateBottomText()`: same for text + build/position circle cylinder, center both as a group
- `updateCircleColor()`: update `circleMaterial.color.setStyle()`
- `applyPreset()`: set color + text, call update functions
- `animate()`: `requestAnimationFrame` loop, `controls.update()`, `renderer.render()`
- `exportSTL()`: `STLExporter.parse(badgeGroup)` (badge only — plates, text, circle; the VRChat base group is excluded from the export), Blob, auto-download
- `handleTopFontChange()` / `handleBottomFontChange()`: `FileReader` → `JSON.parse` → `FontLoader.parse()` → assign font → update text
- `toggleGrid()`: show/hide `gridHelper`
- `toggleVrchatMode()`: fetch `static/models/vrchat-badge-base.obj` → `OBJLoader.parse()` → create mesh with distinct material → add to `vrchatBaseGroup` → auto-position badge group at origin (the base's reference point) → show offset controls
- `positionBadgeOnBase()`: apply X/Y/Z offset to the badge group relative to the base

**Template section:**
- Full-screen relative container
- Floating sidebar card with:
  - Title "EPIC Badge editor"
  - Top text input
  - Top font upload with error alert
  - Dividers
  - Bottom text input
  - Bottom font upload with error alert
  - Circle color picker
  - Status presets buttons
  - **Grid toggle button**
  - **VRChat mode toggle switch**
  - **Badge position offset controls** (X/Y/Z sliders, visible in VRChat mode)
  - Export STL button
- Full-size `<canvas>` behind the card

### `src/routes/layout.css`
- Google Fonts import (Google Sans + Syne)
- `@import 'tailwindcss'`
- `@plugin 'daisyui'`
- `@theme` block defining font families and spacing
- Two `@plugin 'daisyui/theme'` blocks for `dark` and `light` with all radii = 0
- Utility classes: `.font-lang-en`, `.font-lang-th`, `@utility heading-1`, `@utility page-container`

### `src/routes/+layout.svelte`
- Imports `layout.css` and `favicon.svg`
- Renders `<svelte:head>` with favicon link
- `{@render children()}` pattern (Svelte 5)

### `src/app.html`
- Standard SvelteKit HTML shell
- Note: `data-sveltekit-preload-data="hover"` on `<body>`
