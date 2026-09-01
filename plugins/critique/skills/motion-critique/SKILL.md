---
name: motion-critique
description: Review existing animations for timing, easing, purpose, choreography, and performance. Use to judge motion that already exists. Do NOT use to add or improve animation — that's animate/improve-animations.
---

# motion-critique

> **Read first.** Invoke the `frontend-design` skill (plugin `frontend-design@claude-plugins-official`;
> on Linux sandboxes, `/mnt/skills/public/frontend-design/SKILL.md`) — specifically its restraint
> guidance. Extra animation is a primary AI-generated tell. Judge against that standard.

Then load `animate/animation-vocabulary` for shared terminology, so findings are precise ("`ease-in`
on an exit" ) rather than vague ("feels off"). If the `animate` plugin is not enabled, say so and use
the terms defined below anyway.

## Owns
- Judging motion quality across web and Flutter.

## Does not own
- Creating motion → `animate/animate`, `animate/scroll-motion`.
- Fixing existing motion → `animate/improve-animations`.
- Finding where motion is missing → `animate/find-animation-opportunities`.

**This skill produces findings. It does not change code.** If the user says "fix it," hand off to
`animate/improve-animations`.

---

## Output contract

Findings ranked `blocking` / `should-fix` / `nit`. Each states **what · where · why it matters ·
concrete fix with exact values**. End with the single most important change. Never manufacture
findings — "the motion here is already right" is a valid result.

---

## Step 1 — Purpose, before anything else

For every animated element, name its purpose in one of these words: **feedback** (the interface heard
you) · **spatial consistency** (where it came from or went) · **state indication** (a change made
legible) · **preventing a jarring change** · **explanation** (marketing/onboarding only) · **delight**
(rare/first-time only). If none fits, the motion is decoration — a finding, and at high frequency a
`blocking` one.

Then check **frequency**, which sets the bar:

| How often the user sees it | Verdict |
|---|---|
| 100+/day — keyboard shortcuts, command palette, core nav | **Should not animate at all.** `blocking`. |
| Tens/day — hover, list navigation, frequent toggles | Near-imperceptible only, or nothing |
| Occasional — modals, drawers, toasts | Standard animation is correct |
| Rare — onboarding, success, celebration | The delight budget lives here |

Keyboard-initiated actions are a disqualifier, not a judgement call. Raycast has no open/close
animation, and that is the correct design.

Also check **function**: data the user is reading or acting on must not move for style. A decorative
mouse-tracking effect is fine on a marketing page and wrong on a chart in a banking app.

---

## Step 2 — Duration

**Most AI-authored motion is 2–3× too slow.** Check against these, and flag anything over budget
without a written reason:

| Element | Duration |
|---|---|
| Button/press feedback | 100–160ms |
| Tooltips, small popovers | 125–200ms |
| Dropdowns, selects | 150–250ms |
| Modals, drawers | 200–500ms |
| Marketing / explanatory | May be longer |

**UI motion stays under 300ms.** A 180ms dropdown feels more responsive than a 400ms one. Exits are
faster than entrances — an exit at the same duration as its entrance reads as sluggish, because the
user has already decided and is waiting.

---

## Step 3 — Easing

The two most common defects: **`linear` on UI motion** (correct only for continuous or looping
motion — marquees, progress, spinners; elsewhere it reads as mechanical), and **symmetric
`ease-in-out` used reflexively**, or worse **`ease-in` on anything entering or exiting**. `ease-in`
starts slow, delaying exactly the moment the user is watching; `ease-out` at 200ms *feels* faster
than `ease-in` at 200ms.

Judge against: entering and exiting → `ease-out`; moving or morphing while staying on screen →
`ease-in-out`; hover and colour → `ease`; constant motion → `linear`.

Built-in CSS easings are weak. Flag a deliberate animation still using the browser default:

```css
--ease-out:    cubic-bezier(0.23, 1, 0.32, 1);
--ease-in-out: cubic-bezier(0.77, 0, 0.175, 1);
--ease-drawer: cubic-bezier(0.32, 0.72, 0, 1);
```

Springs: bounce above ~0.3 on functional UI is a finding. Reserve bounce for drag-to-dismiss and
deliberately playful moments.

---

## Step 4 — Physicality

- **`scale(0)` entrances.** Nothing in the real world appears from nothing. Start at `0.9–0.97` with
  opacity. `blocking` on a primary surface.
- **`transform-origin: center` on a trigger-anchored popover, dropdown, menu, or tooltip** — it
  should grow from its trigger (`var(--transform-origin)` in Base UI). **Modals are exempt**: they
  are not anchored, so centre is correct.
- **Asymmetric exit path.** A toast entering from the bottom must leave through the bottom. Symmetric
  paths are what make swipe-to-dismiss feel obvious.
- Hardcoded pixel translations where `translateY(100%)` — a percentage of the element's own size —
  would survive a content change.

---

## Step 5 — Choreography

Is there **one orchestrated moment**, or several independent effects firing at once? Scattered
simultaneous animation is noise, and it is the specific failure mode that makes a page feel
AI-generated. The fix is consolidation into one sequence, or deletion of all but one.

Also flag: a stagger outside 30–80ms, or long enough that the last item lands after the user has
moved on; a stagger that blocks interaction; motion fighting motion, such as a page transition
running under a modal entrance.

---

## Step 6 — Interruption

**An animation the user cannot interrupt is a bug.** Ask what happens when the user acts mid-flight —
does it queue, jump, restart from zero, or gracefully re-target?

- **Keyframes on anything rapidly triggered** (toasts, toggles, anything firable twice a second):
  keyframes restart from zero, transitions re-target from the current value.
- Gesture-driven motion without springs loses velocity through an interruption.
- Motion that locks input while it plays is `blocking`.

---

## Step 7 — Performance

**Web:** animating `width`, `height`, `top`, `left`, `margin`, `padding`, or `box-shadow` is a defect
— those trigger layout and paint. `transform` and `opacity` only (`clip-path` is the sanctioned
fourth; `height` is tolerated for accordions, which have no transform equivalent). Also flag
`transition: all`, Motion's `x`/`y`/`scale` shorthands under load instead of the full `transform`
string, a child's transform driven from a CSS variable on the parent, layout reads inside a scroll
handler, and non-passive scroll listeners.

**Flutter:** rebuilding a subtree that could be `AnimatedBuilder`'s `child`; a missing
`RepaintBoundary` around an animating region; `Opacity` where `FadeTransition` would avoid a
`saveLayer`; work inside `build()` during animation; a leaked `AnimationController`. Debug-mode frame
times are meaningless — if that is the evidence offered, say so.

---

## Step 8 — Accessibility of motion

**`prefers-reduced-motion` unhandled is a `blocking` finding on web.** The Flutter equivalent is
`MediaQuery.of(context).disableAnimations`, plus Android's system animation scale.

Reduced motion means *fewer and gentler*, not zero: keep transitions that aid comprehension, remove
translation and scale. A branch that disables an animation and leaves the element in a broken
intermediate state is worse than no branch at all.

Ungated `:hover` motion is a finding — touch fires false hovers on tap. Gate with
`@media (hover: hover) and (pointer: fine)`. Where this overlaps wider accessibility, this skill owns
the motion call and `a11y-review` owns the rest.

---

## Step 9 — Scroll-linked motion

- Hijacking scroll or disabling native scrolling: `blocking`.
- Behaviour on trackpad, wheel, touch, **and** keyboard — all four, not the one you tested.
- Content that only exists after a scroll trigger is a content bug, not a motion nit. Check with JS
  disabled.
- Scroll listeners without `passive: true` and rAF throttling, where `IntersectionObserver` would do.
- Reveals: 200–300ms, 8–24px translate, opacity. Large slide-ins and 600ms-plus fades are the
  standard AI tell — `should-fix`.
- iOS momentum changes scrubbed behaviour. Untested on device is UNVERIFIED, not passing.

---

## Step 10 — Report

Rank by leverage — impact ÷ effort. Anchor every finding to `file:line` with the offending value
quoted, and give the exact replacement value, never an approximation.

**The hardest and most valuable finding to make is "delete this animation." Make it when it is
true.** Where feel genuinely cannot be judged from code — a crossfade, a spring's bounce, a
gesture — say so and name the check (play at 2–5× duration, step frame by frame in the animation
inspector, test gestures on a real device) rather than guessing at a verdict.

## Sources
`frontend-design` (Anthropic); Emil Kowalski's animation philosophy via `review-animations`,
`animate`, and `improve-animations`.
