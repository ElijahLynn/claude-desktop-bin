# Spinner Shapes - Per-Theme Brand-Glyph Replacements

**Companion to** [SPINNER_INJECTION_NOTES.md](SPINNER_INJECTION_NOTES.md) (the injection
design) and `scratchpad/THEME_REFACTOR_CONTRACT.md` (the dual-variant theme contract).

This doc catalogs the actual spinner SVG shapes shipped per theme: what each shape is,
its path data, its color strategy, and how to swap/test one live in the running app.

- **Runtime installer:** `js/spinner_injector.js` (a self-guarding ES5 IIFE).
- **Per-theme specs:** `scratchpad/spinner_specs.json` (merged into each theme's
  `spinner` key at integration by the orchestrator; themes ship `"spinner": null`).
- **How it runs:** the Nim patch prepends `var __CDB_SPINNER_SPEC = <json|null>;` to the
  injector and runs it via the same `wc.executeJavaScript(...)` that injects theme CSS.
  The injector finds the Anthropic 7-point brand-star (`viewBox "0 0 100 100"` +
  a child `<path d>` starting with `m19.6 66.5 19.7-11`) and replaces its `<path>`
  children with the spec's paths, keeping the `<svg>` wrapper so the `fill-current`
  accent and box size are preserved.

> **All shapes:** `viewBox "0 0 100 100"`, centered, ~60-75% of the box, verified
> recognizable at 32px (the `w-8` greeting / thinking-spinner size). Renders checked with
> `rsvg-convert` at 256px and 32px.

---

## Spec format (per theme)

```jsonc
"spinner": {
  "viewBox": "0 0 100 100",          // optional, default "0 0 100 100"
  "match": "m19.6 66.5 19.7-11",     // optional override of the star path signature
  "animation": "spin|bounce|pulse|flare|null",
  "duration": "2s",                  // optional, overrides the stock per-type speed (see below) -- NOT read by `flare`
  "paths": [ { "d": "...", "fill": "#hex" }, ... ]   // omit "fill" => currentColor
}
```

**Color strategy:** a path with **no `fill`** (or `fill: "currentColor"`) inherits the
theme accent through the `<svg>`'s `fill-current` class (which resolves `currentColor`
to `--accent-brand`). An **explicit hex** pins a fixed color - needed only for the
multi-color Mario mushroom. Every single-color shape below omits `fill` so it follows
the theme's brand accent.

**Animation:** adds a `cdb-anim-<spin|bounce|pulse|flare>` class to the replaced `<svg>`.
The keyframes live in the theme CSS (`insertCSS` path), not the injector. `spin` rotates
about the glyph center (`transform-box: fill-box`), `bounce` is a vertical hop, `pulse`
is a flat opacity throb. `flare` is different in kind from the other three: it targets
**individual paths**, not the whole `<svg>` - built for radiating multi-path shapes like
`chai`'s sun. The **first** path (`:nth-child(1)`) is treated as a static center/disc and
gets no animation; each of the next 8 paths (`:nth-child(2)` through `:nth-child(9)`) gets
its own `cdbRayRetract` scale(1 -> .5 -> 1) animation on its **own** hardcoded
duration+delay, so they drift in and out of phase instead of pulsing in unison - a shape
with fewer than 9 paths just leaves the higher `:nth-child` selectors unmatched. Set
`null` to inherit only claude.ai's own motion.

**Duration:** `spin`/`bounce`/`pulse` each have a stock default (1s/.8s/1.2s). The
optional `duration` field (e.g. `"2s"`, `"500ms"`) overrides whichever of those three
types the theme uses - validated against `/^\d+(\.\d+)?m?s$/` in the patch, so a
malformed value silently falls back to the stock default instead of breaking the
generated CSS. Only affects the theme that sets it; every other theme keeps the stock
speed for its type. **`flare` does not read `duration`** - its 8 ray timings are fixed
in the patch (they are inherently 8 independent values, not one number); a future
`rayDurations`-style field could make that themeable if ever needed by a second theme.

---

## The 8 shapes

| Theme | Shape | Paths | Color | Animation |
|-------|-------|-------|-------|-----------|
| `mario` | Mario mushroom | 8 | multi-color (explicit hex) | `bounce` |
| `sweet` | 5-petal blossom | 1 | currentColor (pink accent) | `spin` |
| `nord` | 6-point snowflake | 1 | currentColor | `spin` |
| `catppuccin-mocha` | cat head | 1 | currentColor | `pulse` |
| `catppuccin-macchiato` | cat head | 1 | currentColor | `pulse` |
| `catppuccin-frappe` | cat head | 1 | currentColor | `pulse` |
| `catppuccin-latte` | coffee cup | 1 | currentColor | `pulse` |
| `chai` | 8-ray sun | 1 | currentColor | `flare` |

The three `catppuccin-*` dark variants intentionally **share** the cat-head shape; only
`catppuccin-latte` (the light variant) gets the coffee cup.

---

### 1. `mario` - Mario mushroom (8 paths, multi-color, `bounce`)

The 8-path mushroom straight from [SPINNER_INJECTION_NOTES.md](SPINNER_INJECTION_NOTES.md)
section 5: domed red cap, three white spots, a pale face/stem band, two dark eyes, and a
dark outline drawn first (behind). This is the only shape that does **not** follow the
accent - its colors are baked so it always looks like the canonical mushroom.

Colors: `#E52521` cap, `#FFFFFF` spots, `#FAD9C0` face, `#3A2A1A` outline + eyes.
Build order matters: outline first (behind), then cap, face, spots, eyes on top.

(Full path array: see `scratchpad/spinner_specs.json` -> `mario.paths`, identical
to the 8-path block in the injection notes.)

---

### 2. `sweet` - 5-petal blossom (1 path, currentColor, `spin`)

Five rounded petals radiating from the exact center, plus a center disc. Reads as a
cherry-blossom / flower; follows the theme's pink brand accent. Petals are **rooted at
the center point** (each is `M center, cubic out to tip, cubic back to center`) so they
union with the disc with no inner gaps.

- Construction: 5 petals at 72 deg spacing, first pointing up; tip radius 40, petal
  fatness 0.62, center disc radius 16.
- Color: `currentColor` (no `fill`) -> pink `--accent-brand`.

```
M50 50 C74.8 30 50 10 50 10 C50 10 25.2 30 50 50 Z M50 50 C76.68 67.41 88.04 37.64 88.04 37.64 C88.04 37.64 61.36 20.23 50 50 Z M50 50 C41.69 80.76 73.51 82.36 73.51 82.36 C73.51 82.36 81.82 51.6 50 50 Z M50 50 C18.18 51.6 26.49 82.36 26.49 82.36 C26.49 82.36 58.31 80.76 50 50 Z M50 50 C38.64 20.23 11.96 37.64 11.96 37.64 C11.96 37.64 23.32 67.41 50 50 Z M34 50 a 16 16 0 1 0 32 0 a 16 16 0 1 0 -32 0 z
```

---

### 3. `nord` - 6-point snowflake / crystal (1 path, currentColor, `spin`)

Six spokes at 60 deg spacing, each a thin rectangle from a center hub out to a tip, with
two pairs of small V-branches (at radii 20 and 30) - the classic snowflake silhouette.
Follows the theme accent.

- Construction: 6 spokes (length 40, half-thickness 3.2), branch ticks at 60 deg off
  each spoke (length 9), center hub radius 6. All segments are filled quads/triangles
  concatenated into one path (nonzero fill-rule).
- Color: `currentColor` (no `fill`).

The path is long (~1.5 KB, 31 sub-shapes); see `scratchpad/spinner_specs.json` ->
`nord.paths[0].d` for the literal string. It begins:

```
M50 53.2 L90 53.2 L90 46.8 L50 46.8 Z M67.92 51.2 L72.42 58.99 ...  (31 subpaths) ... M44 50 a 6 6 0 1 0 12 0 a 6 6 0 1 0 -12 0 z
```

---

### 4-6. `catppuccin-mocha` / `-macchiato` / `-frappe` - cat head (1 path, currentColor, `pulse`)

A flat cat-head silhouette: a face circle plus two pointed ear triangles. Follows the
theme accent.

- Construction: face circle radius 27 at `(50,55)`; two ear triangles whose bases sit
  **inside** the dome (outer base low on the flank, inner base near top-center, apex up).
- **Winding gotcha:** the two ears must wind the **same rotational direction as the face
  circle**, or the nonzero fill-rule punches a hole where an ear overlaps the dome (one
  ear comes out clean, the mirror ear shows a white notch). The shipped ears use the
  matching winding (verified: `scratchpad/dbg_cat3_AB.png`, "option B"). If you edit the
  ears, re-render and confirm both connect seamlessly.
- Color: `currentColor` (no `fill`). Falls back to a paw print only if a cat proves hard
  - the cat renders cleanly, so cat it is.

```
M26 45 L49 34 L19 13 Z M74 45 L81 13 L51 34 Z M23 55 a 27 27 0 1 0 54 0 a 27 27 0 1 0 -54 0 z
```

---

### 7. `catppuccin-latte` - coffee cup (1 path, currentColor, `pulse`)

A mug: trapezoid body (wider at top), a flat saucer beneath, an **open-ring** handle on
the right wall, and two slim S-curve steam wisps rising above. Follows the theme accent.

- Construction: body trapezoid (top y=54 half-width 21, bottom y=84 half-width 16);
  saucer trapezoid; handle = an annulus sector (outer arc R=13 bulging right, inner arc
  R=7 returning) attached to the right wall - the **opposite arc sweep flags** cut the
  hole that makes it read as a handle (not a blob); two steam ribbons (closed thin S
  bands) at x=38 and x=50.
- Color: `currentColor` (no `fill`).

```
M23 54 L65 54 L60 84 L28 84 Z M22 87 L66 87 L61 91 L27 91 Z M65 57 A 13 13 0 1 1 65 81 L65 75 A 7 7 0 1 0 65 63 Z M40 50 C45 44 35 40 42 34 C45 28 37 24 40 18 L36 18 C33 24 41 28 38 34 C31 40 41 44 36 50 Z M52 50 C57 44 47 40 54 34 C57 28 49 24 52 18 L48 18 C45 24 53 28 50 34 C43 40 53 44 48 50 Z
```

---

### 8. `chai` - 8-ray sun (9 paths: 1 static disc + 8 independently-animated rays, `flare`)

A center disc with 8 tapered triangular rays radiating at 45deg spacing - a plain sun
glyph. Follows the theme accent (deep spiced-gold in the "chai-light" variant, brighter
gold in "chai-dark"). Unlike every other shape here, this one is **9 separate paths**,
not 1 - the `flare` animation type (see Spec format above) needs each ray as its own
`<path>` so it can retract independently.

- Construction: center disc radius 20; each ray is a 3-point triangle with its two base
  corners on the disc's edge (radius 20, +-9deg either side of the ray's spoke angle) and
  its tip at radius 42 - so the base sits flush against the disc with no seam gap. The
  original single-path version (all 9 subpaths unioned into one `d`, nonzero fill-rule)
  is kept below for reference/copy-paste convenience; the shipped spec splits it into 9
  `paths` entries so `flare` can target them individually.
- Color: `currentColor` (no `fill`), inherited per-path from the `<svg>`'s own fill.
- Animation: `flare` - the disc (path 1) stays static; each ray (paths 2-9) retracts
  toward the center and back out (`scale(1) -> scale(.5) -> scale(1)`) on its own
  duration (3.3s-4.8s) and negative delay, so the 8 rays drift in and out of phase
  instead of pulsing in unison - reads as flares flickering, not a spinning pinwheel and
  not a single uniform pulse.

```
M30 50 a 20 20 0 1 0 40 0 a 20 20 0 1 0 -40 0 Z M46.87 30.25 L50.00 8.00 L53.13 30.25 Z M61.76 33.82 L79.70 20.30 L66.18 38.24 Z M69.75 46.87 L92.00 50.00 L69.75 53.13 Z M66.18 61.76 L79.70 79.70 L61.76 66.18 Z M53.13 69.75 L50.00 92.00 L46.87 69.75 Z M38.24 66.18 L20.30 79.70 L33.82 61.76 Z M30.25 53.13 L8.00 50.00 L30.25 46.87 Z M33.82 38.24 L20.30 20.30 L38.24 33.82 Z
```

---

## How to swap or test a shape live (no rebuild)

The injector is designed for live iteration in the webview DevTools console.

1. Open the claude.ai webview DevTools (right-click -> Inspect on the chat view).
2. Define a spec, then paste the injector body. Pick any shape from
   `scratchpad/spinner_specs.json`, e.g. the cat:

   ```js
   window.__CDB_SPINNER_SPEC = {
     viewBox: "0 0 100 100",
     animation: "pulse",
     paths: [{ d: "M26 45 L49 34 L19 13 Z M74 45 L81 13 L51 34 Z M23 55 a 27 27 0 1 0 54 0 a 27 27 0 1 0 -54 0 z" }]
   };
   // then paste the entire contents of js/spinner_injector.js
   ```

3. Expect a console line `[spinner] matched N glyph(s) on load`. The greeting icon and
   the thinking spinner should switch to the new shape; `currentColor` paths follow the
   theme accent.
4. Re-run a sweep manually: `window.__cdbSpinner.sweep(document.documentElement)`.
   Stop the observer: `window.__cdbSpinner.disconnect()`. Inspect the active spec:
   `window.__cdbSpinner.spec`.

### Authoring / verifying a new shape offline

Shapes were built and visually verified with `scratchpad/build_shapes.py`, which renders
each path to PNG at 256px (inspect) and 32px (real spinner size) via `rsvg-convert`. To
add or tweak a shape: edit the builder, re-render, eyeball the contact sheet, then copy
the path string into `scratchpad/spinner_specs.json`.

```bash
cd scratchpad
python3 build_shapes.py                       # writes shape_<name>.png + shape_<name>_32.png
magick shape_*_32.png +append -filter point -resize 400% sheet.png   # small-size sanity sheet
```

Functional logic of the injector (matcher precision, multi-path, fill handling,
idempotency, no-op-on-null) is covered by `scratchpad/test_injector.js` against a minimal
DOM shim:

```bash
node scratchpad/test_injector.js scratchpad/spinner_specs.json mario js/spinner_injector.js
```

---

## Maintenance notes (version-sensitive)

- **`matched 0` in the console = the logo geometry drifted**, not "feature removed". The
  star path signature (`m19.6 66.5 19.7-11`) is remote-rendered claude.ai geometry this
  repo cannot pin. Per-theme `match` overrides it without a rebuild.
- **Don't switch the injector to `innerHTML`** - it uses `createElementNS` deliberately
  (CSP / Trusted-Types safe for SVG).
- **Editing the cat ears or any union-of-subpaths shape:** re-render and confirm winding;
  a mismatched sub-path winding silently punches a hole under nonzero fill-rule.
- **The mushroom is the only multi-color shape.** Keep every other shape single-path
  `currentColor` so it tracks the theme accent.
