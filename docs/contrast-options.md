# Hard black/white text over a fixed background: options

Goal: every text pixel snaps to pure `#000` or pure `#fff` — no grays — while the
background image remains untouched.

## Why blend modes alone can't do it

Difference, exclusion, and the other separable blend modes are all
piecewise-linear functions of text and backdrop values. Any composition of
piecewise-linear operations yields another piecewise-linear ramp, never a step
function. A threshold is nonlinear, so it must come from a transfer function:

- `filter: contrast(N)` with a large `N`, or
- SVG `feComponentTransfer` with `type="discrete"`.

There is also an ordering constraint: a `filter` runs on an element's own pixels
*before* its `mix-blend-mode` composites the result against the backdrop. So a
filter on the text element itself can never threshold the blend result — the
filter must sit on a layer that already contains the blended output.

## Options

| Option | Structure | Text binary? | Image untouched? | Caveats |
|---|---|---|---|---|
| A. `background-clip: text` | Glyphs become windows onto a second, fixed copy of the image, filtered `invert(1) contrast(20)` | Yes | Yes | Loses `mix-blend-mode` entirely; `background-attachment: fixed` is unreliable on iOS Safari |
| B. Filter the composite | One wrapper around background + text with `contrast(20)` | Yes | No — the photo posterizes to near-black/white too | Also needs restructuring, since a `filter` on the wrapper breaks `position: fixed` inside it (filtered ancestors become containing blocks for fixed-position descendants) |
| C. JavaScript | Drop the blend mode; on scroll, sample the image luminance behind each line or word and toggle `color: #fff` / `#000` | Per line or per word, not per pixel | Yes | Requires code; threshold is fully tunable; works everywhere |

## Option A detail (pure CSS, text-only threshold)

Give each text element the same background image as the fixed background:

```css
.content p {
  background-image: url('IMG_2492.jpeg');
  background-size: cover;
  background-position: center;
  background-attachment: fixed;        /* stays aligned with the real fixed background */
  -webkit-background-clip: text;
  background-clip: text;
  color: transparent;
  filter: invert(1) contrast(20);
}
```

- `background-clip: text` turns the glyphs into windows onto that image copy.
- `background-attachment: fixed` keeps the visible image region viewport-aligned,
  so as the glyphs scroll, each shows the piece of the image it is currently over —
  the same geometry as the `difference` demo.
- `invert(1)` flips polarity: dark backdrop areas render light glyphs.
- `contrast(20)` snaps everything around the 50% luminance point to pure black or
  pure white.
- The threshold point is movable by inserting `brightness(x)` between the two
  filter functions: `invert(1) brightness(1.1) contrast(20)`.

## Recommendation

- Desktop-only demo: Option A — pure CSS, image untouched.
- If iOS Safari must work: Option C — `background-attachment: fixed` is unreliable
  there, so JavaScript thresholding is the safe route.
