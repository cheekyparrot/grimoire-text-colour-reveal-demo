# Alternatives to the mask-image technique: keeping text legible across the horizon

Context: the current demo (`background-clip: text` over a viewport-fixed
mask image) is broken on iOS Safari — WebKit refuses to paint the clipped
background at all when it is combined with `background-attachment: fixed`
(see [`ios-safari-fixed-attachment.md`](ios-safari-fixed-attachment.md)).
This document surveys alternate techniques for the same goal: a fixed
landscape/sky backdrop, with scrolling text that retains contrast as it
crosses the horizon. The mask need not be kept.

## The reframing that opens things up

The current technique needs a full recolored mask image
(`Text-Color-Mask.png`) because it is pixel-general. But the actual
requirement is narrower: the boundary is one **horizontal line at a fixed
viewport Y**. The backdrop is `position: fixed`, `100vw auto`,
`center bottom`, so the horizon never moves, and its viewport Y is
computable in pure CSS from the image's aspect ratio (the same shape of
math as `image-dimensions.txt`). That means most alternatives do not need
per-pixel sampling at all — a scanline-level threshold, or even a
per-element flip, may be visually indistinguishable from the mask demo.

Granularity levels, coarsest to finest:

- **Per element** — the whole paragraph snaps color at once.
- **Per scanline** — each glyph row flips where it crosses the line
  (what a hard-stop gradient or a clip produces).
- **Per pixel** — what the mask image and `mix-blend-mode` produce.

## Options

| # | Technique | JS | Text colors | Reveal granularity | iOS Safari |
|---|---|---|---|---|---|
| 1 | Scroll-driven `view()` timeline color swap | none | any | per element | iOS 26+ |
| 2 | Fixed duplicate layer + `clip-path`, synced by `scroll()` timeline | none (markup duplication) | any | per scanline | iOS 26+ |
| 3 | Gradient text fill + JS `background-position` compensation | small | any | per scanline | works everywhere |
| 4 | IntersectionObserver / scroll classes | small | any | per element or per word | works everywhere |
| 5 | Text stroke / shadow scrim instead of color change | none | single color | n/a (different effect) | works everywhere |
| 6 | `mix-blend-mode: difference` (the original technique) | none | black/white only | per pixel | works everywhere |
| 7 | Fixed-mask twin text | none (markup duplication) | any | per pixel | iOS 26+ |

### 1. Scroll-driven animations — the option that did not exist at the last survey

Scroll-driven animations shipped in Safari 26 / iOS 26 (September 2025),
with Chrome/Edge support since 115 (July 2023). Earlier conclusions of the
form "iOS ⇒ JavaScript" predate this. Drop the mask entirely and animate
`color` as each paragraph crosses the horizon:

```css
.content p {
  color: #1b1b1b; /* fallback: over-sky color */
  animation: flip linear both;
  animation-timeline: view();
  animation-range: entry 0% exit 100%;
}
@keyframes flip {
  0%, 50%      { color: #1b1b1b; }
  50.01%, 100% { color: #f4f1e6; }
}
```

Advantages over the mask path:

- It never touches `background-attachment`, the property iOS parses but
  ignores.
- It degrades cleanly: browsers without `animation-timeline` drop the
  declaration and keep the static fallback color.
- `@supports (animation-timeline: view())` is a *real* feature test —
  unlike `background-attachment`, where parse success and behavior
  diverge.

Caveats:

- The flip is per paragraph, not per scanline — a straddling paragraph
  snaps all at once, which reads differently from the mask demo.
- The exact flip point (which keyframe percentage corresponds to the
  horizon Y) depends on paragraph height, so `animation-range` needs
  per-paragraph tuning. The discrete snap gives tolerance for being
  slightly off.

### 2. Fixed duplicate layer with `clip-path` — per-scanline fidelity, no JS

This is option 3 in `ios-safari-fixed-attachment.md` ("restructure with a
fixed-position layer"), now buildable cleanly because the boundary is one
horizontal line:

- Base scrolling text in the land-side color.
- A `position: fixed` viewport-sized wrapper (`overflow: hidden`) with
  `clip-path: inset(0 0 calc(100% - var(--horizon-y)) 0)` — showing only
  the region above the horizon — containing a **duplicate** of the
  content in the sky-side color.
- Sync the duplicate with `animation-timeline: scroll(root)`, animating
  `transform: translateY(calc(-100% + 100vh))` on the clone stack. The
  clone stack's own height equals the document height, so
  `-100% + 100vh` is exactly "scrolled by the same amount."

Glyph rows flip exactly at the line, as in the mask demo; full color
freedom; no `background-attachment` anywhere.

Costs: duplicated markup (or a tiny JS clone), and the same iOS 26+
support window as option 1. Wrap in `@supports` and fall back to a static
color.

### 3. Gradient fill + JS compensation — option 2 of the iOS doc, simplified

The iOS diagnosis established that `background-attachment: scroll`
combined with `background-clip: text` is a paint path WebKit renders
fine — the break is only the intersection with `fixed`. So keep
`background-clip: text`, but replace the mask **PNG with a hard-stop
`linear-gradient`** (sky-side color above the horizon fraction, land-side
below), and let a scroll handler update `background-position-y` so the
stop tracks the viewport horizon. Because the threshold is one line, the
compensation math is trivial compared to compensating a whole image.

Per-scanline granularity, any colors, works on every iOS. Carries over
the known caveat from the iOS doc: momentum-scroll lag between finger
and fill, mitigable with `requestAnimationFrame` and `visualViewport`
events, but not eliminable.

### 4. IntersectionObserver / scroll-driven class toggling

Option C of `contrast-options.md`. Measure the horizon Y once; on scroll,
compare each paragraph's (or line's, or word's) bounding rect against it
and toggle a class. Works on all iOS, full color freedom, threshold fully
tunable — but per-element or per-line granularity only, and it is JS.

### 5. The design cheat: do not change color at all

The real goal is *contrast*, not color change. A single text color with
`-webkit-text-stroke` + `paint-order: stroke fill` (supported in Safari),
or a soft `text-shadow` scrim, stays legible over both sky and land.
Works everywhere with zero machinery. It changes the aesthetic though —
no reveal. Worth a quick visual trial before committing to any of the
above, to confirm the reveal itself is wanted rather than mere legibility.

### 6. `mix-blend-mode: difference` — status unchanged

Works on iOS, per pixel, but black/white only, and that limit is
inherent: any separable blend of a *constant* text color is
piecewise-linear in the backdrop (the argument in
[`contrast-options.md`](contrast-options.md)), and `difference` with pure
black/white is the only member that guarantees contrast against both
halves. Colored text under `difference` yields photo-negative hues, not
controlled colors. So: acceptable fallback, not the path to colored text.

### 7. Fixed-mask twin text (option 2 refined: the mask as the wrapper's clip)

Option 2 as written above has two accuracy limits, both accepted at the time:

- The horizon in the image is not remotely a horizontal line, so a
  straight-line `clip-path` (or a gradient stop in option 3) approximates
  it as one Y — wrong at every ragged edge.
- Responsive design will likely mean more than one backdrop configuration
  (different images, crops, anchorings, displayed aspect ratios), so the
  single `--horizon-y` formula becomes a per-breakpoint derivation. As
  geometric approximations, those variants top out at "roughly coincide."

The escape hatch is that the only thing iOS Safari actually breaks is one
specific intersection: `background-attachment: fixed` combined with
`background-clip: text`. Option 2's fixed wrapper needs neither property.
It is `position: fixed`, which iOS honours, and anything painted on a
fixed element is inherently viewport-pinned without
`background-attachment`. So the wrapper's clip does not have to be a
geometric line at all — it can be `mask-image`, using the same
`100vw auto` / `center bottom` geometry rules as the backdrop:

- Sky-colour copy inside the fixed wrapper, wrapper masked by an **alpha
  coverage mask** (opaque above the horizon, transparent below) instead
  of `clip-path: inset(...)`.
- The mask shares geometry with the backdrop exactly the way
  `Text-Color-Mask.png` does today — a recolour of the same scene,
  carrying coverage instead of colour. Converting the existing asset is a
  near-trivial edit.

This restores per-pixel accuracy over an arbitrarily ragged horizon: the
boundary the text crosses is the mask's actual silhouette, not a line.
It also dissolves the responsive-calculation problem: if the backdrop
switches image or crop per breakpoint, the mask switches with it — a
burden the current demo already pays (mask and backdrop must remain
geometric twins). In CSS terms, one custom-property set per breakpoint
(`--mask-image`, `--mask-size`, `--mask-position`) mirrored from the
backdrop's own values; there is no horizon-Y formula to derive, because
the geometry is data, not calculation.

Remaining costs are those of option 2: the `scroll()`-timeline lockstep
sync (iOS 26+), an `@supports`-guarded fallback to a static colour, and
duplicated markup — which need not be hand-written:

**The tiny JS clone.** Duplication is a construction problem, not a
runtime one, so it can run once at page load while the `scroll()` timeline
does all the syncing:

```js
const twin = document.querySelector('.content').cloneNode(true);
twin.setAttribute('aria-hidden', 'true');
document.querySelector('.twin-overlay').appendChild(twin);
```

- The twin stays registered with the original without any per-frame
  code: it inherits the same stylesheet, and the fixed overlay is
  `inset: 0` — the same containing-block width the original lays out
  against — so line breaks come out identical. The clone stack's height
  also matches the document's scroll height, which is what makes the
  `translateY(calc(-100% + 100vh))` end keyframe correct without
  measuring anything.
- The overlay should be `pointer-events: none; user-select: none` so
  selection and clicks only ever hit the base copy; the twin carries
  `aria-hidden` so screen readers do not read the text twice.
- Building the overlay in JS makes the no-JS case a clean fallback for
  free: browsers without JS just see the base single-colour text — no
  half-painted layer to guard against. The cost is that "no JavaScript"
  becomes "a few lines at load, no scroll listeners" rather than
  literally none.
- The one genuine regression: if the text ever becomes dynamic (CMS,
  client-side rendering, a font swap that reflows after the clone), the
  twin silently goes stale — re-clone on mutation or font load. For a
  static demo, a non-issue.

The lockstep sync is the fragile part — if the two copies ever reflow
differently (font loading, width), they drift apart.

Implementation findings (Safari 26.6, October 2026): the sync is exact on
load, but two WebKit behaviors had to be worked around in the demo's
load-time script. First, the scroll timeline's range and the transform's
percentage base are captured when the animation binds and are not
re-read after layout changes, so a window resize permanently desyncs
the twin until the animation is re-bound (toggling `animation-name`
after the resize settles). Second, without `will-change: transform` the
animated clone shares a compositing path with the masked overlay and
its frame updates can fall out of step with the document scroll,
flashing the base copy's glyph edges through the twin mid-scroll; its
own composited layer fixes that. Both are invisible to
`getComputedStyle` diagnostics — computed values read correct while
pixels diverge — so variant testing against live rendering was needed
to isolate them.

## Recommendation

For a demo where the effect is the point: **option 1** as the primary —
pure CSS, colored text, graceful degradation, `@supports`-testable — with
a decision on the legacy-iOS fallback: either `mix-blend-mode: difference`
black/white (strictly better than a static color there, and per-pixel) or
a plain static color.

If per-scanline fidelity across the straddling moment is what makes the
current demo, option 2 is the pure-CSS version and option 3 is the
works-everywhere version; when the horizon is ragged or the backdrop
configuration varies by breakpoint, option 7 — fixed-mask twin text — is
option 2's per-pixel-exact refinement.

One open question: what fraction of real iOS visitors sits below iOS 26
— devices capped at iOS 18 cannot get scroll-driven animations, so if
old-iOS fidelity matters, that pushes toward option 3 or a hybrid.
