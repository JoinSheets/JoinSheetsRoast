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
#2's border and #4/#8's more saturated light tone hold up better there. No
pick made yet -- waiting on review.
