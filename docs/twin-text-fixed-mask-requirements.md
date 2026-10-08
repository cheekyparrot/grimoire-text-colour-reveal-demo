# The option 7 mask: what it is and how to make it

Reference: [technique-alternatives.md](technique-alternatives.md), option 7
("Fixed-mask twin text"). This note expands the mask asset itself — its
required qualities and the conversion from the existing asset.

## What the mask is

An **alpha coverage mask**: the sky/land division lives entirely in the
PNG's alpha channel, not in its colours. When used as CSS `mask-image`,
the browser reads **alpha only** (the default `mask-mode: match-source`),
so:

- **Opaque (alpha 1)** wherever the twin's sky-coloured text should show —
  the sky side of the horizon.
- **Transparent (alpha 0)** wherever the base layer's land-coloured text
  should show through — the land side.
- Colour channels are irrelevant; solid white over the opaque region is
  the convention. A black-and-white *opaque* image would do nothing —
  greyscale only matters under `mask-mode: luminance`, which option 7 does
  not use.
- Partial alpha at the boundary is fine and arguably desirable: it
  blends the two text colours at soft edges instead of aliasing between
  them.

## The qualities it must have

1. **A true ragged silhouette, not a line.** Per-pixel accuracy is the
   entire point of option 7 — the boundary the text crosses is the mask's
   actual shape. The region division must trace the real sky/land edge of
   the scene, inherited exactly from whatever boundary the source asset
   encodes.
2. **Geometric twin of the backdrop.** Same scene, same crop, same pixel
   dimensions — never resized or re-cropped independently. It will be
   painted with the mirrored rules (`mask-size: 100vw auto`,
   `mask-position: center bottom`, `mask-repeat: no-repeat`), so mask
   pixel N lands on backdrop pixel N. Same caveat as today: on very
   tall/narrow viewports neither image reaches the top — consistent with
   the backdrop, so nothing new.
3. **A format that carries alpha.** PNG (or WebP); JPEG is out.
4. **A per-breakpoint variant whenever the backdrop has one.** Each
   backdrop image/crop gets its own mask, wired through `--mask-image` /
   `--mask-size` / `--mask-position`.
5. **A same-origin URL.** `mask-image: url()` loads with CORS, unlike
   `background-image`. Over `file://` (opaque "null" origin) every
   browser silently blocks the mask and the overlay paints nothing —
   the demo must be served over HTTP for the mask to appear at all.

## How to make it

The option 7 write-up calls it a near-trivial edit because
`Text-Color-Mask.png` already encodes the boundary — in colour rather
than coverage. Conversion is a two-step recolour:

1. Take the existing mask (or its working file).
2. Replace the sky-side colour with opaque white, and the land-side
   colour with transparency — i.e., map the two colour regions straight
   onto alpha 1 and alpha 0, keeping dimensions and edge softness
   untouched. Export as PNG.

The one thing to confirm before converting: that the existing asset's
boundary is the traced ragged silhouette, not a straight-line split —
the new mask can only be as accurate as the boundary it inherits.
