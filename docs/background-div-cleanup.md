# Cleanup note: the `.background` div is not a leftover

Context: `index.html` no longer uses `mix-blend-mode`. The current technique is
two viewport-fixed layers that are pixel-aligned:

1. **Backdrop** — `.background`, a fixed full-viewport div painting
   `Background.jpeg` at `100vw auto, center bottom`, with `z-index: -1`.
   The negative z-index is load-bearing: `.content` is unpositioned, so
   without it the positioned backdrop div would paint on top of the
   paragraphs' clipped-to-text background fill and the text would vanish
   behind the image.
2. **Masked text** — `.content p` sets `color: transparent` and fills the
   glyphs with `Text-Color-Mask.png` via `background-clip: text`, with
   `background-attachment: fixed` so scrolling pans the text across the
   mask instead of moving the mask with it.

Both layers use identical sizing and positioning rules (`100vw auto`,
`center bottom`, viewport-fixed), so the color a glyph shows is the mask
image's pixel sitting on top of the corresponding backdrop pixel. The mask
is a recolored duplicate of the scene; this duplicated geometry is what
replaced the old blend-mode compositing.

## The consequence

The `.background` div is easy to mistake for a leftover from the
blend-mode version. It is not safe to delete:

- The glyph fill keeps working without it (text stays colored by the
  mask), so deleting the div does not immediately "break" anything in an
  obvious way.
- But the visible scene disappears, leaving `body`'s `background: #000`.
  The demo loses the context that makes the reveal read as "text colored
  by the landscape."

If the div ever needs to go, the replacement is a refactor, not a removal:
move the backdrop onto `body` itself with `background-attachment: fixed`
and identical `background-size` / `background-position`, then delete the
div. Until that is wanted, keep `.background` as-is.
