---
worth: yes
where: src/dithering/error-diffusion.ts:33
added: 2026-10-25
---
# error diffusion carries its error in sRGB, CIELAB or OKLab, never in linear light

`errorDiffuse` keeps each pixel and the error it passes on in the chosen error space: `rgb` (sRGB
values), `cielab` or `oklab`. The eye averages dots in linear light, so with few levels the mid-tones
come out too light. On the reTerminal E1001 in black and white (measured black 26, white 213), a 128
grey gets 54% white dots in the `rgb` space, which the eye averages to about 163. Carried in linear
light it gets 31%, which averages to 128. With four greys the steps are smaller, but the error points
the same way. DBS already works in linear light (`src/dithering/dbs.ts`, YyCxCz), so Floyd-Steinberg
and DBS put the same picture at different lightness, and the preview shows it, as it too averages in
linear light (`src/processing/preview.ts`).

aitjcize/epaper-image-convert does this for its grey palettes, and by default (`processor.js`,
`applyErrorDiffusionDither`, `linearLight`): the pixel and its error are linear RGB 0..1, the nearest
level is still chosen in sRGB or Lab (the working value encoded back first), and the error is the
working value minus the chosen level, both linear.

Where it would go:

- A `linear` value of `ColorSpace` (`src/types.ts:33`), allowed as the error space only. A nearest
  match in linear light is not perceptual, so it pairs with an `rgb`, `cielab` or `oklab` distance.
- `convertToSpace` returns linear RGB for it (`srgbToLinear` is in `src/processing/colorspace.ts`). The
  clamp before the lookup (around line 117) takes 0..1. The step to the distance space (around line
  135) converts only from `rgb` now, so `linear` encodes back to sRGB (`linearToSrgb`) and then goes on
  as `rgb` does.
- Six algorithms go through `errorDiffuse`: Floyd-Steinberg, Atkinson, Jarvis, Stucki, Sierra and
  Burkes. Dizzy and Riemersma keep their own loops and would need the same change; Knox ignores the
  spaces, as it always works in OKLab.
- The colour presets in `index.html` (around line 280) are `<error>_<distance>` pairs; add
  `linear_cielab` and `linear_oklab`, and their labels in `src/main.ts` (`colorSpaceLabel`).
- Open: whether grey palettes (the `grayscale*` groups) should pick it by default, as
  epaper-image-convert's do. Colour palettes keep what they have.

Correct is also darker: mid-tones lose the lift the wrong average gave them. On a panel in a dim
spot, as the user's is, expect to raise Exposure or Shadow boost after switching. That is a choice
for where the panel hangs, not a fault in the calibration.

The repo has no tests. A check: dither a grey ramp with Floyd-Steinberg in `linear_cielab` and with
DBS, then compare the average linear light of each column. They should match within a few percent,
which `rgb_rgb` does not.
