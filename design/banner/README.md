# Banner drafts

Four draft variants of a homepage hero image: JavaScript as the main motif
(the course dissects the JoinSheets codebase, written in JS), with a gear
badge as a "bonus" accent nodding at the course's parallel Rust track --
a generic gear, not a copy of the official Rust logo mark. All four use the
project's own brand palette (`#2b2b2e` dark / `#86868a` gray / `#d98f2b`
orange), the same three colors as `static/images/logo.svg`.

Not part of the Hugo build (this directory lives outside `static/`,
`content/`, `config/`, `layouts/`, and `themes/`) -- purely for review.

- `gallery.html` -- open locally in a browser to see all four side by side,
  on both a light and a dark background.
- `banner-NN-<concept>-js.svg` -- individual files.

## Variants

- `banner-01-squares-js.svg` -- the brand mark's own overlapping rounded
  squares (enlarged), with "J"/"S" lettered inside two of them.
- `banner-02-braces-js.svg` -- curly braces `{ }` framing "JS".
- `banner-03-codetag-js.svg` -- a `</>` code tag over a "JAVASCRIPT" wordmark.
- `banner-04-bigsquares-js.svg` -- two solid squares, bold "J"/"S", a larger
  and more prominent gear badge.

All four: 480x480, transparent background, gear badge bottom-right (01, 02)
or top-right (03) or bottom-center (04).

## Current pick

`banner-04-bigsquares-js.svg` -- picked. Wired up as
`static/images/homepage-banner.svg`, used on the course homepage
(`content/_index.ru.md`/`_index.en.md`) in a two-column
`{{< block "grid-2" >}}` layout (image left, annotation + button right).
