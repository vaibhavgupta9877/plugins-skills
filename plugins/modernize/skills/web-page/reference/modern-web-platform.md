# Modern web platform pass

Current platform features that make a page feel 2026-native **by deleting JavaScript or hacks**, not
by adding effects. Rules for every row:

- **Check support against the project's `browserslist` / stated targets** (and caniuse/MDN at the time
  of the run) — support moves; do not trust this table's notes blindly.
- **Progressive enhancement only.** Guard with `@supports`, feature detection, or a fallback that is
  already acceptable. A missing feature must never break layout, content, or the primary action.
- Adopt a feature only where the page has the problem it solves. This is not a checklist to max out.

## Layout

| Feature | Replaces | Fallback |
|---|---|---|
| Container queries (`@container`, `cqi` units) | Viewport media queries on reusable components | Single-column default layout |
| `subgrid` | Hand-matched row heights across cards | Independent grids |
| `:has()` | JS that toggles parent classes (form-valid, card-with-media) | Neutral default style |
| `dvh` / `svh` / `lvh` | `100vh` mobile bug, JS height fixes | `min-height: 100vh` declared first |
| `clamp()` fluid type and space | Breakpoint jumps in font size | Built in (fully supported) |
| Logical properties (`margin-inline`, `inset-block`) | LTR-only spacing | Built in |
| `scrollbar-gutter: stable`, `aspect-ratio` | Scrollbar and media layout shift | Built in |

## Typography

| Feature | Use | Note |
|---|---|---|
| `text-wrap: balance` (headings) / `pretty` (body) | Removes orphans and ragged headline breaks | Ignored where unsupported — harmless |
| Variable fonts, self-hosted WOFF2, subset | One file, full weight/width range; fewer requests | Framework font loader (e.g. `next/font`) generates metric-matched fallbacks |
| `font-variant-numeric: tabular-nums` | Numbers that change in place | Built in |

## Colour

| Feature | Use | Fallback |
|---|---|---|
| `oklch()` tokens | Perceptually even ramps; predictable dark-mode variants | Hex/rgb declared first, or build-time conversion |
| `color-mix()`, relative colour syntax | Hover/tint/alpha variants derived from one token | Precomputed token values |
| `color-scheme` + `light-dark()` | Theme-aware tokens without duplicated selectors | `[data-theme]` / media-query token sets |
| Wide gamut accent under `@media (color-gamut: p3)` | Richer accent on capable screens | sRGB accent |
| `prefers-contrast`, `forced-colors` | High-contrast and Windows forced-colour modes stay usable | Visible borders on controls by default |

## Motion and transitions

| Feature | Replaces | Fallback |
|---|---|---|
| View Transitions (same-document `startViewTransition`; cross-document `@view-transition { navigation: auto }`) | Heavy SPA transition libraries; jarring route swaps | Instant navigation (already correct) |
| Scroll-driven animations (`animation-timeline: view()` / `scroll()`) | Scroll listeners and some GSAP for simple progress/reveal effects | Static end state or `IntersectionObserver` |
| `@starting-style` + `transition-behavior: allow-discrete` | JS mount/unmount animation for dialogs, popovers, toasts | Instant show/hide |
| `linear()` easing | Spring-like curves without a JS spring library | A cubic-bezier ease-out |
| `prefers-reduced-motion` on every non-trivial motion | — | Required, not optional |

Everything here stays subject to `animate` gates: purpose first, one signature moment, fast exits.

## Components

| Feature | Replaces | Note |
|---|---|---|
| `<dialog>` + `showModal()` | Hand-rolled modal `div`s with broken focus trapping | Native focus trap, `Esc`, top layer, `::backdrop` |
| Popover API (`popover`, `popovertarget`) | JS menus/tooltips for show/hide and light-dismiss | Top layer; no z-index wars |
| CSS anchor positioning | JS positioning libraries for tooltips/menus | Keep the library where support is insufficient |
| `<details name="…">` | JS exclusive accordions | Non-exclusive accordion |
| `inert` | Manual `tabindex=-1` sweeps behind overlays | Built in on modern targets |
| `field-sizing: content` | Auto-growing textarea scripts | Fixed rows |

## Loading and responsiveness

| Feature | Use |
|---|---|
| `fetchpriority="high"` on the LCP image only; `loading="lazy"` below the fold | LCP |
| AVIF/WebP via the framework image pipeline, correct `sizes` | Bytes |
| `content-visibility: auto` + `contain-intrinsic-size` on long off-screen sections | Render cost on long pages |
| Speculation Rules (`<script type="speculationrules">`) for likely next pages | Near-instant navigation where supported; ignored elsewhere |
| `scheduler.yield()` (with `setTimeout` fallback) in long tasks | INP |

## Retire on sight

`100vh` heroes · scroll listeners for reveals · JS masonry for simple grids · `z-index: 9999` ·
custom `div` modals · autoplay carousels · icon fonts · full-library imports for one helper ·
`@import` chains in CSS · layout animated with `top/left/height`.
