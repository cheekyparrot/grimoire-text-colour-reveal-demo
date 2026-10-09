# Text Colour Reveal Demo

A small demo of a CSS **text colour reveal** technique: paragraphs scroll
up a page with a fixed landscape backdrop, flipping from the land-side
colour to the sky-side colour as they cross the horizon — per pixel,
following the horizon's actual ragged silhouette. The only JavaScript is
a few lines at page load; the browser's scroll-driven animations do all
the syncing. No blend modes, no scroll listeners.

Safari/iOS 26 added CSS scroll-driven animations, in particular the `animation-timeline` property (2023 for Edge and Chrome) making this CSS-only technique possible. (JavaScript was used to sync text across twin layers, but that's ancillary.) It's pretty new tooling and not fully supported. Pure fallback on Firefox.

The technique is only valuable if it can be used responsively, which remains unproven. But it's interesting enough to merit publication.

This demo has seen limited testing (passing) on desktop Safari and Chrome, 
iOS 26, simulated iOS 26, and simulated Android. 

Serve the directory over HTTP and open the page from `http://localhost`:

```sh
python3 -m http.server 8000
```

or host it anywhere static (it works as a
[GitHub Pages](https://pages.github.com/) site as-is).

**Do not open `index.html` directly via `file://`** — the page will be
blank. `mask-image: url()` is a CORS-mode resource load, and a
`file://` page has an opaque origin, so every browser (Safari, Chrome,
Firefox) silently blocks the mask images; a blocked mask masks the
whole overlay out — and with both overlays gone, the invisible-ink base
is all that remains, so nothing is visible at all. Gradients are
unaffected (they are not resources), which is why this can masquerade
as a Safari-only bug. The old `background-clip: text` demo did not
have this constraint: background images are not CORS-checked. Serving
over HTTP is load-bearing, not a convenience.

## The four-layer structure

The effect is built from four layers, the mask-bearing three
pixel-aligned with each other:

1. **The backdrop** — a fixed, full-viewport `.background` div painting
   `Background.jpeg` (`100vw auto`, `center bottom`) behind everything,
   with `z-index: -1` so it stays beneath the unpositioned content.
2. **The base copy** — `.content`, scrolling normally. While it is the
   fallback (no JS, no scroll-driven animations) it paints the
   land-side colour (`#FFFAD8`); once the twins are live it becomes
   **invisible ink** — still carrying layout (the document height the
   sync math depends on), selection, and screen-reader text, but
   painting nothing.
3. **The sky twin** — `.twin-overlay.sky`, a `position: fixed`,
   `inset: 0` overlay containing a JavaScript clone of `.content` in
   the sky-side colour (`#925006`, brown against the bright sky),
   painted through `Sky-Coverage-Mask.png`: an **alpha coverage mask**
   opaque over the sky.
4. **The land twin** — `.twin-overlay.land`, the same arrangement in
   the land-side colour, painted through `Land-Coverage-Mask.png`,
   opaque over the land.

The two masks are exact complements — their alpha channels sum to 1.0
at every pixel — so each colour of text is confined to its own side of
the horizon and no copy can ever paint the wrong colour in the wrong
region. That confinement is the point: the twins no longer need
frame-exact glyph coverage to look right. Scrolling pans the text up
the viewport; an `animation-timeline: scroll(root)` keyframe animation
translates both clones by the same amount, so they stay in lockstep
with zero per-frame code.

## Pixel alignment

The masks and the backdrop use identical sizing and positioning rules —
`100vw auto`, `center bottom` — on identically-sized boxes (all
`inset: 0`, so on iOS they track the live viewport as the URL bar
expands and collapses), so they are **pixel-aligned**: the boundary the
text crosses is the masks' actual silhouette sitting exactly on the
backdrop's own horizon, ragged edges included.

`Sky-Coverage-Mask.png` and `Land-Coverage-Mask.png` are coverage twins
of the scene in `Background.jpeg`: same scene, same dimensions, same
crop, carrying alpha instead of colour — one opaque over the sky, the
other over the land, exact complements (see
[`docs/twin-text-fixed-mask-requirements.md`](docs/twin-text-fixed-mask-requirements.md)
for the full asset requirements). If the backdrop ever switches image or
crop per breakpoint, the masks switch with it, via the
`--sky-mask-image` / `--land-mask-image` / `--mask-size` /
`--mask-position` custom properties.

## Key CSS

```css
.twin-overlay {
  position: fixed;
  inset: 0;
  overflow: hidden;
  mask-size: 100vw auto;
  mask-position: center bottom;
  mask-repeat: no-repeat;
}
.twin-overlay.sky  { mask-image: var(--sky-mask-image); }
.twin-overlay.land { mask-image: var(--land-mask-image); }
.twin-overlay .content {
  animation: twin-scroll linear both;
  animation-timeline: scroll(root);
}
@supports (animation-timeline: scroll()) {
  .twins .content p { color: transparent; }
}
@keyframes twin-scroll {
  to { transform: translateY(calc(-100% + 100dvh)); }
}
```

The end keyframe is what makes the sync measurement-free: the clone
stack's height equals the document's scroll height, so translating it by
`-100% + 100dvh` at the end of the scroll range scrolls it by exactly the
amount the page itself has scrolled. The `@supports` rule turns the
in-flow base into invisible ink, but only while scroll-driven animations
exist — and only when the script's `twins` class says the clones were
built, so every failure path keeps the visible fallback.

## The tiny JS clone

```js
if (CSS.supports('animation-timeline', 'scroll()')) {
  const base = document.querySelector('.content');
  for (const overlay of document.querySelectorAll('.twin-overlay')) {
    const twin = base.cloneNode(true);
    twin.setAttribute('aria-hidden', 'true');
    overlay.appendChild(twin);
  }
  document.documentElement.classList.add('twins');
}

// WebKit captures the scroll timeline's range and the transform's
// percentage base when the animation binds, and does not re-read them
// after a resize — so re-bind each clone once the window settles.
let rebind;
addEventListener('resize', () => {
  clearTimeout(rebind);
  rebind = setTimeout(() => {
    for (const twin of document.querySelectorAll('.twin-overlay .content')) {
      twin.style.animationName = 'none';
      void twin.offsetWidth;
      twin.style.animationName = '';
    }
  }, 150);
});
```

It runs once at page load, plus a one-shot re-bind per clone whenever
the window is resized (see the findings bullet below). Everything is
gated on the feature test: the clones are built only if scroll-driven
animations exist, and the `twins` class — which turns the base into
invisible ink — is added last, so any failure above leaves the visible
base as the fallback. The clones inherit the same stylesheet and lay
out against the same viewport width (the overlays are `inset: 0`), so
line breaks come out identical to the original. The overlays are
`pointer-events: none; user-select: none` so selection and clicks only
ever hit the base copy, and the twins carry `aria-hidden` so screen
readers do not read the text twice.

Without JavaScript there are no clones, no overlay content — the page
is just the base text in a single colour.

## Details worth knowing

- **iOS Safari works here, by construction.** The technique never
  touches `background-attachment: fixed` combined with
  `background-clip: text` — the one intersection WebKit refuses to paint
  (see [`docs/ios-safari-fixed-attachment.md`](docs/ios-safari-fixed-attachment.md)
  for the original diagnosis). The backdrop is `position: fixed` and the
  twin is clipped by `mask-image`, both of which iOS honours.
- **Support window.** The reveal needs scroll-driven animations (Safari
  26 / iOS 26+, Chrome/Edge 115+). Browsers without it — and browsers
  without JavaScript — keep the static base colour:
  `@supports not (animation-timeline: scroll())` hides the overlays so
  stationary twins never sit over the wrong text. The feature test is
  a real one, unlike `background-attachment` on iOS, where parse
  success and behavior diverge.
- **The lockstep sync no longer has to be frame-exact.** Earlier
  versions needed the brown twin to cover the cream base glyph-for-glyph
  every frame, and any per-frame divergence flashed cream glyph edges
  through the brown mid-scroll. The complementary masks remove that
  requirement structurally: no copy paints outside its own region, so
  divergence between the twins can only show as a brief vertical tear
  where a glyph straddles the horizon, never as the wrong colour in
  the wrong region. What still needs care, measured on Safari 26.6:
  WebKit captures the scroll timeline's range and the transform's
  percentage base when an animation binds and never re-reads them, so
  a window resize permanently desyncs the twins until each animation
  is re-bound — hence the resize handler in the script above, which
  re-binds both. `will-change: transform` gives each clone its own
  composited layer so the twins' frame updates commit together. If
  the text ever becomes dynamic (CMS, client-side rendering),
  re-clone on mutation or font load.
- **Selection and semantics live on the invisible base.** The twins
  are `aria-hidden` and unselectable (`pointer-events: none;
  user-select: none`), so reading and copy/paste are carried by the
  in-flow copy whose glyphs are transparent — selecting it paints the
  highlight on invisible text, but selects and copies correctly.
- The negative `z-index` on `.background` is load-bearing: `.content` is
  unpositioned, so without it the positioned backdrop div would paint on
  top of the paragraphs and the text would vanish.
- The first paragraph gets a viewport-based bottom margin (`30svh`) so
  the second always starts below the fold, no matter how the first wraps.
- **Viewport-anchored geometry must measure the live viewport, never
  `vh`.** On iOS, the URL bar expands and collapses the viewport that
  the scroll timeline's range and fixed boxes measure against, while
  `100vh` stays pinned to the large viewport height. Two `vh`-based
  measurements broke before this rule was established: the backdrop's
  `height: 100vh` anchored its image below the visible bottom, out of
  step with the mask at the top of the scroll; and the twin's end
  keyframe (`-100% + 100vh`) travelled a large-viewport distance
  across a live-viewport timeline range, desyncing the two copies in
  proportion to scroll depth. Both are fixed by the live measure —
  `inset: 0` for the backdrop's box, `100dvh` in the keyframe — and
  the content's paddings use `svh`, so the document height the clone
  mirrors stays stable across toolbar states.
- Beyond the viewport, the page falls back to `body`'s black
  background — the masks and the backdrop share this behaviour. On
  tall, narrow viewports the bottom-anchored image does not reach the
  top of the viewport, and text there vanishes entirely, because
  nothing paints outside the masks' coverage. That is intentional:
  region confinement means no text ever floats over a
  backdrop-less band.

## History

This demo evolved from an earlier `mix-blend-mode` experiment, then a
`background-clip: text` mask-image approach (invisible text on iOS
Safari), and now implements **fixed-mask twin text** from
[`docs/technique-alternatives.md`](docs/technique-alternatives.md): the
mask repurposed as the clip of a fixed twin-text overlay, with the old
`Text-Color-Mask.png` colour mask as the source asset for the coverage
mask's silhouette. The complementary-mask refinement — one twin per
side of the horizon, each confined by an exact-complement coverage
mask, with the in-flow base as invisible ink — came out of the
frame-timing glitches of the single-twin version: confinement removed
the frame-exactness requirement the glitches fed on.
