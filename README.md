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

**Do not open `index.html` directly via `file://`** — you will see only
the base cream text and no reveal. `mask-image: url()` is a CORS-mode
resource load, and a `file://` page has an opaque origin, so every
browser (Safari, Chrome, Firefox) silently blocks the mask image; a
blocked mask masks the whole overlay out. Gradients are unaffected
(they are not resources), which is why this can masquerade as a
Safari-only bug. The old `background-clip: text` demo did not have
this constraint: background images are not CORS-checked.

## The three-layer structure

The effect is built from three layers, the top two pixel-aligned with
each other:

1. **The backdrop** — a fixed, full-viewport `.background` div painting
   `Background.jpeg` (`100vw auto`, `center bottom`) behind everything,
   with `z-index: -1` so it stays beneath the unpositioned content.
2. **The base copy** — `.content`, scrolling normally, in the land-side
   colour (`#FFFAD8`, cream over the dark ground).
3. **The sky-coloured twin** — `.twin-overlay`, a `position: fixed`,
   `inset: 0` overlay containing a JavaScript clone of `.content` in the
   sky-side colour (`#925006`, brown against the bright sky), painted
   through `Sky-Coverage-Mask.png`: an **alpha coverage mask** that is
   opaque above the horizon and transparent below.

Wherever the mask is opaque the twin's sky-coloured text shows; wherever
it is transparent the base text shows through it. Scrolling pans the base
text up the viewport; an `animation-timeline: scroll(root)` keyframe
animation translates the twin by the same amount, so the two copies stay
in lockstep with zero per-frame code.

## Pixel alignment

The mask and the backdrop use identical sizing and positioning rules —
`100vw auto`, `center bottom` — on identically-sized boxes (both
`inset: 0`, so on iOS they track the live viewport as the URL bar
expands and collapses), so they are **pixel-aligned**: the boundary the
text crosses is the mask's actual silhouette sitting exactly on the
backdrop's own horizon, ragged edges included.

`Sky-Coverage-Mask.png` is a coverage twin of the scene in
`Background.jpeg`: same scene, same dimensions, same crop, carrying alpha
instead of colour (see
[`docs/twin-text-fixed-mask-requirements.md`](docs/twin-text-fixed-mask-requirements.md)
for the full asset requirements). If the backdrop ever switches image or
crop per breakpoint, the mask switches with it, via the
`--mask-image` / `--mask-size` / `--mask-position` custom properties.

## Key CSS

```css
.twin-overlay {
  position: fixed;
  inset: 0;
  overflow: hidden;
  mask-image: var(--mask-image);
  mask-size: 100vw auto;
  mask-position: center bottom;
  mask-repeat: no-repeat;
}
.twin-overlay .content {
  animation: twin-scroll linear both;
  animation-timeline: scroll(root);
}
@keyframes twin-scroll {
  to { transform: translateY(calc(-100% + 100dvh)); }
}
```

The end keyframe is what makes the sync measurement-free: the clone
stack's height equals the document's scroll height, so translating it by
`-100% + 100dvh` at the end of the scroll range scrolls it by exactly the
amount the page itself has scrolled.

## The tiny JS clone

```js
const twin = document.querySelector('.content').cloneNode(true);
twin.setAttribute('aria-hidden', 'true');
document.querySelector('.twin-overlay').appendChild(twin);

// WebKit captures the scroll timeline's range and the transform's
// percentage base when the animation binds, and does not re-read them
// after a resize — so re-bind once the window settles.
let rebind;
addEventListener('resize', () => {
  clearTimeout(rebind);
  rebind = setTimeout(() => {
    twin.style.animationName = 'none';
    void twin.offsetWidth;
    twin.style.animationName = '';
  }, 150);
});
```

It runs once at page load, plus a one-shot re-bind whenever the window
is resized (see the findings bullet below). The clone inherits the same
stylesheet and lays out against the same viewport width (the overlay is
`inset: 0`), so line breaks come out identical to the original. The
overlay is `pointer-events: none; user-select: none` so selection and
clicks only ever hit the base copy, and the twin carries `aria-hidden`
so screen readers do not read the text twice.

Without JavaScript there is no clone, no overlay content — the page is
just the base text in a single colour.

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
  `@supports not (animation-timeline: scroll())` hides the overlay so a
  stationary twin never sits over the wrong text. The feature test is a
  real one, unlike `background-attachment` on iOS, where parse success
  and behavior diverge.
- **The lockstep sync is the fragile part.** Measured on Safari 26.6:
  on a fresh load the twin is pixel-exact, but WebKit captures the
  scroll timeline's range and the transform's percentage base when the
  animation binds and never re-reads them, so a window resize
  permanently desyncs the twin until the animation is re-bound — hence
  the resize handler in the script above. Separately, `will-change:
  transform` on the clone gives it its own composited layer; without
  it, the twin's frame updates can fall out of step with the document
  scroll, and the base copy's glyph edges flash through the twin
  mid-scroll. If the text ever becomes dynamic (CMS, client-side
  rendering), re-clone on mutation or font load.
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
- Beyond the viewport, the page falls back to `body`'s black background —
  the mask and the backdrop share this behaviour, so the twin is never
  asked to cover text the backdrop is not covering.

## History

This demo evolved from an earlier `mix-blend-mode` experiment, then a
`background-clip: text` mask-image approach (invisible text on iOS
Safari), and now implements **option 7 — "fixed-mask twin text"** from
[`docs/technique-alternatives.md`](docs/technique-alternatives.md): the
mask repurposed as the clip of a fixed twin-text overlay, with the old
`Text-Color-Mask.png` colour mask as the source asset for the coverage
mask's silhouette.
