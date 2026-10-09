# The fixed-mask twin text masks: what they are and how to make them

Reference: the "fixed-mask twin text" method in
[technique-alternatives.md](technique-alternatives.md). This note
expands the mask assets themselves — their required qualities and the
conversion from the existing asset.

## What the masks are

Two **alpha coverage masks**, exact complements: the sky/land division
lives entirely in the PNGs' alpha channels, not in their colours. When
used as CSS `mask-image`, the browser reads **alpha only** (the default
`mask-mode: match-source`), so:

- `Sky-Coverage-Mask.png` — **opaque (alpha 1)** over the sky,
  transparent over the land: it confines the sky-coloured twin to the
  sky side of the horizon.
- `Land-Coverage-Mask.png` — the inverse: **opaque** over the land,
  transparent over the sky: it confines the land-coloured twin to the
  land side.
- The two masks' alpha channels sum to exactly 1.0 at every pixel, so
  the twins' regions tile the viewport with neither gap nor overlap
  at the ragged boundary.
- Colour channels are irrelevant; solid white over the opaque region is
  the convention. A black-and-white *opaque* image would do nothing —
  greyscale only matters under `mask-mode: luminance`, which the
  fixed-mask twin text method does not use.
- Partial alpha at the boundary is fine and arguably desirable: it
  blends the two text colours at soft edges instead of aliasing
  between them.

## The qualities it must have

1. **A true ragged silhouette, not a line.** Per-pixel accuracy is the
   entire point of the fixed-mask twin text method — the boundary the
   text crosses is the mask's actual shape. The region division must
   trace the real sky/land edge of the scene, inherited exactly from
   whatever boundary the source asset encodes.
2. **Geometric twin of the backdrop.** Same scene, same crop, same pixel
   dimensions — never resized or re-cropped independently. Both masks
   are painted with the mirrored rules (`mask-size: 100vw auto`,
   `mask-position: center bottom`, `mask-repeat: no-repeat`), so mask
   pixel N lands on backdrop pixel N. On very tall/narrow viewports
   neither image reaches the top of the viewport; text there is
   intentionally unpainted — region confinement means nothing floats
   over the backdrop-less band.
3. **A format that carries alpha.** PNG (or WebP); JPEG is out.
4. **A per-breakpoint variant whenever the backdrop has one.** Each
   backdrop image/crop gets its own pair of masks, wired through
   `--sky-mask-image` / `--land-mask-image` / `--mask-size` /
   `--mask-position`.
5. **A same-origin URL.** `mask-image: url()` loads with CORS, unlike
   `background-image`. Over `file://` (opaque "null" origin) every
   browser silently blocks the mask and the overlay paints nothing —
   the demo must be served over HTTP for the mask to appear at all.

## How to make it

The technique-alternatives write-up calls it a near-trivial edit because
`Text-Color-Mask.png` already encodes the boundary — in colour rather
than coverage. Conversion is a two-step recolour, plus one channel
inversion for the land twin:

1. Take the existing mask (or its working file).
2. Replace the sky-side colour with opaque white, and the land-side
   colour with transparency — i.e., map the two colour regions straight
   onto alpha 1 and alpha 0, keeping dimensions and edge softness
   untouched. Export as `Sky-Coverage-Mask.png`.
3. Produce `Land-Coverage-Mask.png` by negating the sky mask's alpha
   channel and nothing else (RGB stays white). The pair must remain
   exact complements — alpha summing to 1.0 at every pixel — so the
   twins' regions tile with neither gap nor overlap.

The one thing to confirm before converting: that the existing asset's
boundary is the traced ragged silhouette, not a straight-line split —
the new mask can only be as accurate as the boundary it inherits.
