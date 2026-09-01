# Timing table

Values, not vibes. Never approximate a curve because it looks familiar — take it from here, from
[easing.dev](https://easing.dev/), or from the codebase's existing tokens.

---

## Duration by element

| Element | Duration | Note |
|---|---|---|
| Button / press feedback | 100–160ms | Any slower and the press feels laggy |
| Hover state | 100–150ms | Colour and small transforms only |
| Tooltips, small popovers | 125–200ms | Skip the animation entirely on subsequent hovers |
| Dropdowns, selects, menus | 150–250ms | 180ms feels more responsive than 400ms |
| Accordion / disclosure | 200–300ms | Height is tolerated here |
| Modals, drawers, sheets | 200–500ms | Larger surface earns more time |
| Route / page transition | 300–400ms | Above 400ms the app feels slow, not smooth |
| Toast enter | 200–300ms | Exit ~150ms |
| Marketing / explanatory | Longer is allowed | It is being watched, not used |

**UI motion stays under 300ms.** Past 500ms, write down the reason.

**Exits are faster than entrances.** Roughly 0.6–0.8× the entrance. On entry the user is being
oriented; on exit they have already decided and are waiting.

---

## Easing by situation

| Situation | Easing | Why |
|---|---|---|
| Entering | `ease-out` | Fast start = immediately visible response |
| Exiting | `ease-out` (faster duration) | Never `ease-in` — it delays the moment being watched |
| Moving / morphing on screen | `ease-in-out` | Element is present throughout; symmetric reads correct |
| Hover, colour | `ease` | Cheap and unobtrusive |
| Continuous / looping | `linear` | The only correct use of linear |

```css
--ease-out:    cubic-bezier(0.23, 1, 0.32, 1);
--ease-in-out: cubic-bezier(0.77, 0, 0.175, 1);
--ease-drawer: cubic-bezier(0.32, 0.72, 0, 1);
```

Built-in `ease-out` is too weak for a deliberate animation. Use the custom curve.

---

## Stagger

| Group size | Delay per item |
|---|---|
| 2–5 items | 50–80ms |
| 6–12 items | 30–50ms |
| More than ~12 | Don't stagger — animate the container once |

Cap total stagger time at ~300ms. If the last item lands after the user has moved the pointer, the
stagger is decoration that costs responsiveness. Stagger must never block interaction with items
that have already arrived.

---

## Delay

Delays are almost always wrong on functional UI — they read as lag. The exceptions:

- **Tooltip open**: 300–700ms delay to prevent accidental activation. Then **skip both the delay and
  the animation** for adjacent tooltips while one is already open; it makes the whole toolbar feel
  faster.
- **Hover-intent menus**: a short delay before closing, so a diagonal mouse path doesn't dismiss it.
- **Loading indicators**: delay the spinner ~200–500ms, so fast responses never flash one.

---

## Perceived performance

Perception is the product; measured milliseconds are the input.

- `ease-out` at 200ms *feels* faster than `ease-in` at 200ms — the movement is visible immediately.
- A 180ms select feels more responsive than a 400ms one even though both are "fast".
- Instant feedback on press, before the network resolves, makes the same request feel faster.
- A skeleton shaped like the real layout beats a spinner: it starts answering the question "what am I
  waiting for" immediately.
- A faster-spinning spinner makes an identical load feel shorter.

---

## Never ship

| Never | Instead |
|---|---|
| `transition: all` | Name the exact properties |
| `transform: scale(0)` entrance | `scale(0.95)` + `opacity: 0` |
| `ease-in` on a UI element | `ease-out`, or the strong custom curve |
| Built-in `ease-out` on a deliberate animation | `cubic-bezier(0.23, 1, 0.32, 1)` |
| Animation on a keyboard shortcut or 100+/day action | No animation |
| UI duration over 300ms with no stated reason | 150–250ms |
| `transform-origin: center` on a trigger-anchored popover | `var(--transform-origin)` — modals exempt |
| Keyframes on toasts, toggles, rapidly-triggered elements | CSS transitions |
| Animating `width`/`height`/`margin`/`top`/`left` | `transform` / `opacity` |
| Motion `x`/`y`/`scale` props under load | Full `transform` string |
| Ungated `:hover` motion | `@media (hover: hover) and (pointer: fine)` |
| Missing `prefers-reduced-motion` | Gentler variant, not zero |
| Everything entering at once | 30–80ms stagger |
