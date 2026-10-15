# Responsive extension tiles

A design for making the backdrop scene work across viewport aspect
ratios without cropping, distorting, or shipping an oversized canvas:
keep the photographic scene as a core image, and supplement it with
extension tiles — extra background layers that fill whatever gap the
core leaves at the current aspect ratio.

- On a **wide (landscape) viewport**, the core is supplemented with
  side tiles to the right (the region under the text column).
- On a **tall (portrait) viewport**, the core is supplemented with a
  sky fill above and, if the anchoring changes (see below), a tile
  below.

Extension tiles need not be photographic. A tile whose content is
flat — solid sky, uniform ground — is procedural: a `background-color`
or a `linear-gradient` layer costs nothing to download or decode.
The historical "sky box" (removed in `1b791ba`, see
`git log`) is the degenerate case: its gradient was a flat fill whose
only complexity came from being an image layer that required geometry.

Related terms: the closest established analogues are **nine-slice
scaling** (a protected core with edges that adapt to the container —
but by stretching a single asset rather than compositing several),
**set extension** in VFX, and **outpainting** in generative imagery.
"Skirt" (from 3D terrain) names the below-the-horizon tile.

## Why tiles instead of one extended asset

Baking the extension into the assets avoids all CSS complexity, but
on a portrait phone viewport (~390×844) a landscape image (say 3:2)
renders only ~260 CSS px tall: the strip to fill is more than twice
the image itself, and an adequate baked extension roughly triples the
canvas area. Flat regions compress to almost nothing on the wire —
tens of KB against a ~1 MB backdrop — but the costs that scale with
pixels do not:

- **Decode work** scales with pixels, not bytes: a mostly-flat
  4032×9000 bitmap costs roughly 3× the decode of the scene alone.
- **Texture memory** scales the same way once rasterized
  (~49 MB → ~145 MB at 4 bytes/px before browser downsampling), held
  permanently by a fixed backdrop, on a mobile GPU.

A procedural layer avoids both entirely: zero bytes, negligible
decode, a tiny composited strip. Photographic tiles are only worth
their cost where the scene detail must visibly continue.

## The lockstep rule

Every backdrop layer must be mirrored in **both** coverage masks with
identical geometry, in every breakpoint regime. The masks' only job is
to align the colour-switch boundary (the horizon) with the scene; any
layer that exists in one of the three elements (`.background`,
`.twin-overlay.sky`, `.twin-overlay.land`) and not the others breaks
that alignment. What that means per tile:

- **A tile that crosses the horizon** (e.g. a side tile containing
  both sky and field) needs real coverage-mask twins: three art
  assets per tile, horizon-accurate.
- **A tile entirely on one side of the horizon** needs no mask file.
  A below-horizon ground tile is all "land": the land mask gets a
  procedural opaque layer sized like the tile; the sky mask gets
  nothing, because uncovered mask area is already masked out. Same in
  reverse for a sky-side tile.
- **The sky fill** is fully procedural: `background-color` on the
  backdrop (or a gradient, if the sky needs one), plus one procedural
  opaque layer in the sky mask sized to the strip. Zero files.

So the expected file-count explosion does not happen: real
coverage-twin assets are confined to horizon-crossing tiles.

Coverage-mask transparency caveat: a procedural underlayer shows
through anywhere the layers above it are translucent, so the core and
any photographic tiles must be opaque where they overlap a procedural
fill.

## Costs

1. **Seams.** Every internal join is a potential hairline at
   fractional device pixel ratios (2×/3× rounding) and an
   art-direction problem: a photographic continuation must match the
   core's edge, and repeated or mirrored tiles read as visible
   symmetry. Budget a ~2px overlap at each join (the historical sky
   box used `+ 2px` for exactly this), consistently across all three
   layer lists.
2. **Per-regime configuration, three times.** Loading side tiles only
   on wide viewports and vertical fills only on tall ones means
   `aspect-ratio` media queries switching whole
   `background-image` / `mask-image` / size / position lists on
   `.background`, `.twin-overlay.sky`, and `.twin-overlay.land`. The
   lockstep invariant is enforced by hand in each regime.
3. **`100vw` vs the fixed boxes.** On desktop with classic scrollbars,
   `100vw` is wider than the `inset: 0` box the layers actually paint
   in. Today the three layers are consistently offset together, but
   per-tile `calc()` positions computed against those boxes will
   surface the discrepancy. Prefer `100%` of the inset box over
   `100vw` for tiled geometry.
4. **Viewport units must be live.** Any strip-sized layer must use
   `dvh` (or the inset box itself), never `100vh`: `vh` pins to the
   large viewport on iOS while the fixed boxes track the live one —
   the desync `index.html`'s comments already document in two places.

## Anchoring decision to settle first

A tile **below** the core only becomes necessary if the design
abandons bottom-anchoring. Today the core's bottom edge is always
flush with the viewport bottom, so a tall viewport needs nothing below
it. Adding a lower tile implies the scene (horizon) should be framed
consistently across orientations instead of pinned to the bottom edge
— a legitimate goal, but a bigger change than tiling: it moves the
horizon relative to the fold, so the reveal choreography
(`70svh` padding, `30svh` first-paragraph gap) needs review per
regime too.

Bottom-pinned is the smaller design: the whole tall-viewport case
collapses to a procedural sky fill.
