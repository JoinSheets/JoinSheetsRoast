# Banner color-scheme variants

Round 2 of banner exploration: the composition (two squares + circular badge
with the crab) is settled (see `../banner/`), this round only plays with
color -- square gradients/borders and badge background/ring -- while reusing
the exact same geometry (squares, crab shape) as the current
`static/images/homepage-banner.svg`.

General direction: tone increases left to right (left square lighter, right
square darker) -- the opposite of the current live banner, which goes
dark-to-orange. One variant (`05`) reproduces the current live banner
unchanged, for side-by-side comparison. Several variants deliberately reuse
`#d98f2b`, the plain orange used as the gear's backing circle in the very
first (pre-crab) banner draft.

Not part of the Hugo build (outside `static/`, `content/`, `config/`,
`layouts/`, `themes/`) -- purely for review.

- `gallery.html` -- open locally in a browser to see all 8 side by side, on
  both a light and a dark background.
- `banner-NN-<concept>.svg` -- individual files, 480x480, same crab/square
  geometry throughout.

## Variants

1. `banner-01-mono-fade.svg` -- neutral light gray -> dark, no border, white
   ring badge.
2. `banner-02-mono-bordered.svg` -- same fade as #1, but with a thin
   semi-transparent stroke around each square (keeps an edge on a light page
   background) and an orange badge ring.
3. `banner-03-warm-amber.svg` -- warm cream -> dark, amber badge ring.
4. `banner-04-orange-to-charcoal.svg` -- left square is the original
   "gear background" orange (`#d98f2b`) from the first banner draft, right
   square a deep charcoal.
5. `banner-05-classic-current.svg` -- the current live banner's colors
   (dark -> orange), kept here unchanged for comparison.
6. `banner-06-cool-neutral.svg` -- muted/desaturated fade, gray badge ring
   (neither orange nor dark) for a quieter accent.
7. `banner-07-dark-badge.svg` -- badge colors inverted (dark fill, light
   ring, brighter crab) -- meant to hold up if the page ever runs in a dark
   theme.
8. `banner-08-duotone-orange.svg` -- both squares stay within the orange
   family (pale -> brownish-orange), a single-hue duotone instead of a
   neutral/orange split.

Caveats noticed while reviewing: the lightest left squares in #1, #2 and #6
have low contrast against a plain white page background (edges read soft);
#2's border and #4/#8's more saturated light tone hold up better there.

## Pick: squares

`banner-04-orange-to-charcoal.svg` picked for the squares (left `#d98f2b`
orange fading to `#e4a23f`, right deep charcoal fading to `#1c1b1e`).

## Round 3: badge ring + inner background (`badge-NN-*.svg`, `gallery-badge.html`)

Follow-up on the picked squares: the original badge had two concentric
circles (an outer ring circle and a smaller inner circle for the
background), which left a plain, fairly wide white band between them around
the crab. These 8 variants drop that and use a single circle throughout (one
fill, one stroke) to remove the band, and vary the ring color and the inner
fill independently. Squares are fixed to the picked orange-to-charcoal
scheme in every variant.

1. `badge-01-gradient-orange-ring.svg` -- soft white-to-peach radial
   gradient fill, ring in the left square's orange.
2. `badge-02-flat-white-red-ring.svg` -- flat white fill, ring in a deep
   rust-red (darker than the crab itself, for contrast against white).
3. `badge-03-flat-peach-gold-ring.svg` -- flat peach fill, gold ring.
4. `badge-04-flat-white-no-ring.svg` -- flat white fill, no ring at all.
5. `badge-05-gradient-soft-charcoal-ring.svg` -- same gradient fill as #1,
   ring in a softer charcoal (lighter than the square's own dark tone),
   thinner stroke.
6. `badge-06-flat-gray-gray-ring.svg` -- flat light gray fill, ring in the
   brand's original gray (`#86868a`) -- the quietest, least saturated option.
7. `badge-07-flat-orangetint-ring-gradient-sq.svg` -- flat pale-orange fill,
   ring itself gradient orange-to-charcoal, echoing the squares' own gamut.
8. `badge-08-flat-white-ring-gradient-orange-red.svg` -- flat white fill,
   ring gradient from the square's orange to the crab's red, bridging both
   accent colors already in the composition.

Caveat: #6's gray ring is faint against a plain white page background (same
issue as the earlier gray-accent round-2 variant).

## Pick: badge

`badge-03-flat-peach-gold-ring.svg` picked (flat peach fill, gold ring) --
wired up as `../../static/images/homepage-banner.svg`, the live course
homepage banner.
