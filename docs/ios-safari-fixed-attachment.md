# `background-attachment: fixed` on iOS Safari

## The problem

iOS Safari does not support `background-attachment: fixed`. It silently
treats the value as `scroll`, so a background declared fixed actually
attaches to the element's own box and moves with it when the page
scrolls. This is a long-standing behaviour (not a recent regression),
and because the property *parses* fine everywhere, `@supports` cannot
detect it.

## What breaks in this demo

The text colour reveal depends on the mask fill
(`Text-Color-Mask.png` on `.content p`) being pinned to the viewport:

- On compliant browsers, scrolling pans the paragraphs across a
  viewport-fixed mask, so glyph colour tracks position in the scene.
- On iOS Safari, the mask attaches to each paragraph instead. Each
  paragraph carries its own bottom-aligned slice of the image around,
  the panning reveal is lost, and the fill no longer stays pixel-aligned
  with the backdrop.

The backdrop is unaffected: `.background` holds still via
`position: fixed`, which iOS Safari supports fine. That split is what
makes the bug visible here — the scene stays put while the text fill
drifts with it, instead of the two staying locked together.

## Confirmed diagnosis (iOS Simulator, 2026-10-07)

The failure reproduces in the Xcode iOS Simulator, so the Safari Web
Inspector could drive single-property toggles on the live page. The
observed symptom is worse than the drift described above: the
paragraphs are completely invisible. `color: transparent` leaves the
glyphs with nothing to paint, because WebKit drops the clipped
background's paint entirely when `background-attachment: fixed` is
combined with `-webkit-background-clip: text`.

The toggle matrix localises the failure to exactly that combination:

| Test (one property changed) | Result | Reading |
|---|---|---|
| `background-attachment: fixed` disabled | text paints, in a white slice of the mask | the clip trick itself works; only `fixed` breaks it |
| `-webkit-background-clip: text` disabled (with `fixed`) | mask paints as an unclipped box background that scrolls with the content; text stays transparent | the mask loads and paints under `fixed`, and its scrolling confirms `fixed` is behaving as `scroll` |
| `color: hotpink` (with `fixed` + clip) | glyphs paint pink | glyph layout and rendering are fine |

Every ingredient paints individually; only the intersection of
`fixed` attachment and text clipping produces no paint.

Note that the inspector's computed panel still reports
`background-attachment: fixed` on the broken browser — the declaration
parses fine, so the computed value cannot be trusted as a detection
signal, and `@supports` is equally blind. Detection has to be
behavioural.

Two consequences for the options below:

- The symptom to fix is invisible text, not just a drifting fill. The
  README note should say iOS visitors see the fallback colour, not a
  degraded version of the effect.
- Option 2's compensation only helps if the mask runs with
  `background-attachment: scroll` and the `background-position` is
  updated by hand, which its description already implies. Compensating
  while keeping `fixed` would feed a paint path WebKit never renders.

## Possible solutions

### 1. Static fallback colour (simplest, most robust)

Detect the broken behaviour in JavaScript and, when detected, give the
paragraphs a plain `color` and drop the mask fill:

```js
// Behavioural detection: a fixed-attached background on a
// tall element should be offset by more than the viewport.
// If it isn't, fixed attachment is being ignored.
function supportsFixedAttachment() {
  const probe = document.createElement('div');
  probe.style.cssText =
    'position:absolute;top:0;height:200vh;' +
    'background:url(data:) fixed no-repeat;';
  document.body.appendChild(probe);
  const ok = probe.getBoundingClientRect().top === 0;
  document.body.removeChild(probe);
  return true; // replace with a real offset measurement for your case
}
```

The sketch above shows the shape of a behavioural probe; a production
version needs a measurable image and a real comparison. On failure,
set `color` and remove the `background-clip`/`background-image` rules —
`background-clip: text` with `color: transparent` would otherwise make
the text invisible.

Trade-off: iOS visitors get legible text but not the effect.

### 2. JavaScript compensation (preserves the effect)

Keep the mask as a scroll-attached background, but on every scroll frame
update each paragraph's `background-position` so the image lands where a
viewport-fixed attachment would have put it — i.e. offset by the
paragraph's distance from the viewport top. The pixel-alignment with the
backdrop is then restored by hand.

Caveats:

- iOS fires scroll events asynchronously during momentum scrolling, so
  updates can visibly lag behind the finger. Updating in a
  `requestAnimationFrame` loop and using
  `visualViewport` events reduces, but does not eliminate, the lag.
- It reintroduces JavaScript into an otherwise JS-free demo, and the
  compensation code must respect the same `100vw auto` /
  `center bottom` sizing rules to keep the layers aligned.

### 3. Restructure with a fixed-position layer

Move the mask image into a `position: fixed` element (which iOS honours)
and derive the text colour from it. In practice that means duplicating
the text: a fixed overlay holds the mask image clipped to glyph shapes,
and the scrolling copy is hidden underneath — or an SVG with a fixed
`clipPath`/`pattern` in user space. This can match the original effect
pixel-for-pixel but is the largest change to the demo's structure and
the hardest to keep in sync with the flowing text.

## Recommendation

For a public demo, option 1 (graceful fallback to a static colour) is
the right default: it is robust, keeps the demo JavaScript-free on
compliant browsers, and iOS visitors still read the text. Option 2 is
worth attempting only if the effect itself is the point on mobile; test
on a real device, because desktop Safari's responsive design mode does
not reproduce the attachment behaviour. (The Xcode iOS Simulator does
reproduce it, including the invisible-text variant, so it is a valid
platform for confirming a fix; only momentum-scroll behaviour there
differs enough to require a final check on hardware.)

Whichever route is chosen, the README's "Details worth knowing" should
note the iOS limitation and what visitors on iOS will see.
