# Text Colour Reveal Demo

> **This branch is the frozen `background-attachment: fixed` method.**
> Development of the current technique now happens on `main`, which uses
> scroll-driven animations and complementary masks instead of a
> viewport-fixed backdrop. This branch is preserved as a separate,
> independent line of the project.

A small, dependency-free demo of a CSS **text colour reveal** technique:
paragraphs scroll up the page while their glyphs act as windows onto a
viewport-fixed mask image, which dictates the text's colour directly.
No JavaScript, no blend modes — one HTML file.

Open `index.html` in a browser, or host it anywhere static (it works as
a [GitHub Pages](https://pages.github.com/) site as-is).

## The two-layer structure

The effect is built from two viewport-fixed layers that are pixel-aligned
with each other:

1. **The backdrop** — a fixed, full-viewport `.background` div painting
   `Background.jpeg` (`100vw auto`, `center bottom`) behind everything,
   with `z-index: -1` so it stays beneath the unpositioned content.
2. **The masked text** — `.content p` sets `color: transparent` and fills
   the glyphs with `Text-Color-Mask.png` using `background-clip: text`.
   `background-attachment: fixed` pins that fill to the viewport, so the
   image does not scroll with the text; scrolling pans the paragraphs
   across it instead.

## Pixel alignment

Both layers use identical sizing and positioning rules — `100vw auto`,
`center bottom`, viewport-fixed. That makes them **pixel-aligned**: the
colour a glyph shows at any viewport position is the mask image's pixel
sitting exactly on top of the corresponding backdrop pixel.

`Text-Color-Mask.png` is a recoloured duplicate of the scene in
`Background.jpeg`. Because the two images share geometry, the text
samples its colour from the same point of the scene it covers — without
`mix-blend-mode`, compositing filters, or JavaScript.

## Key CSS

```css
.content p {
  background-image: url('Text-Color-Mask.png');
  background-size: 100vw auto;
  background-position: center bottom;
  background-repeat: no-repeat;
  background-attachment: fixed;
  -webkit-background-clip: text;
  background-clip: text;
  color: transparent;
}
```

## Details worth knowing

- **iOS Safari fails hard.** It ignores `background-attachment: fixed`,
  and when that declaration is combined with `background-clip: text`,
  WebKit refuses to paint the clipped background at all: the paragraphs
  are completely invisible, not just misaligned with the backdrop. The
  backdrop itself is unaffected (`position: fixed` works fine). Confirmed
  on the Xcode iOS Simulator by single-property toggling — see
  [`docs/ios-safari-fixed-attachment.md`](docs/ios-safari-fixed-attachment.md)
  for the full diagnosis and the fix options. Until one of those fixes
  lands, iOS visitors see no text.
- The negative `z-index` on `.background` is load-bearing: `.content` is
  unpositioned, so without it the positioned backdrop div would paint on
  top of the paragraphs' clipped-to-text fill and the text would vanish.
- The first paragraph gets a viewport-based bottom margin (`30svh`) so
  the second always starts below the fold, no matter how the first wraps.
- Viewport units use `svh` so the reveal stays stable when mobile browser
  chrome expands and collapses.
- Beyond the viewport, the page falls back to `body`'s black background.

## History

This demo evolved from an earlier `mix-blend-mode` experiment, which
derived text colour by compositing the text against the backdrop. The
current version replaces blending with the pixel-aligned mask-image
approach described above.
