---
name: improve-animations
description: Fix motion that already exists but feels wrong — sluggish, janky, distracting, or uninterruptible. Do NOT use to judge quality without changing it — that's critique/motion-critique.
---

# improve-animations

> **Read first.** Invoke the `frontend-design` skill (plugin `frontend-design@claude-plugins-official`;
> on Linux sandboxes, `/mnt/skills/public/frontend-design/SKILL.md`) and follow its restraint guidance.

## Owns
- Refining motion that already exists, and deleting motion that should not.

## Does not own
- Judgement-only output with no changes → `critique/motion-critique`.
- New motion → `animate`.
- Finding where motion is missing → `find-animation-opportunities`.

---

## Step 1 — Diagnose before changing

Load `animation-vocabulary` and **name the actual defect** before touching a value. "Feels wrong" is
not a diagnosis, and tweaking durations until something feels better produces motion nobody can
maintain.

Recon first: the motion library in use, the existing easing and duration tokens, and the frequency
tier of the element. **Extend the codebase's tokens — adding a parallel system is a defect**, even
when your values are better than theirs.

Then state the defect in the vocabulary's terms: "`ease-in` on the drawer exit at 450ms", "keyframes
on a toast, so rapid triggers restart from zero", "animating `height` on the sidebar".

---

## Step 2 — The five defects that account for most bad motion

Work these first; they are where the leverage is.

**1. Too slow.** The most common defect by a wide margin, because the instinct that produced it was
"make it smooth". Halve it. Check against the budget: press 100–160ms, dropdown 150–250ms, modal
200–500ms, and UI generally under 300ms. Then make the exit faster than the entrance — around
0.6–0.8× — because on exit the user has already decided.

**2. Wrong easing direction.** `ease-in` on an exit, or on anything. It starts slow and delays the
exact moment the user is watching. Entering and exiting → `ease-out`; on-screen movement →
`ease-in-out`; continuous → `linear`, and nothing else takes `linear`. Replace weak built-ins with
`cubic-bezier(0.23, 1, 0.32, 1)`.

**3. Animating non-compositor properties.** `width`, `height`, `top`, `left`, `margin`, `padding`,
`box-shadow` trigger layout and paint every frame. Convert to `transform` and `opacity`; use FLIP for
a layout change that has no direct transform equivalent. Also fix `transition: all`, which animates
properties you never intended.

**4. No interruption handling.** The element feels stuck because the animation must finish. Swap
keyframes for transitions on anything rapidly triggered — transitions re-target from the current
value, keyframes restart from zero. Use springs for anything gesture-driven so velocity carries
through the interruption.

**5. Too many simultaneous independent animations.** Noise, and the specific thing that makes a page
feel AI-generated. Consolidate into one orchestrated sequence, or cut all but one. **One orchestrated
moment lands harder than scattered effects.**

---

## Step 3 — Flutter-specific jank

Symptoms first, because the fixes are different:

- **UI-thread jank** (expensive `build()`): animating a widget above a rebuild-heavy subtree instead
  of passing that subtree as `AnimatedBuilder`'s `child`. This is the single highest-impact Flutter
  animation fix — the `child` is built once and reused every frame.
- Work inside `build()` during animation: allocations, list transformations, `MediaQuery` lookups.
  Hoist them out.
- **Raster-thread jank** (expensive painting): a missing `RepaintBoundary` around the animating
  region, so its neighbours repaint too.
- `Opacity` forcing a `saveLayer` — use `FadeTransition` or `AnimatedOpacity`.
- A leaked `AnimationController` (no `dispose()`) — a real memory bug, not just a perf one.
- **Profile in profile or release mode.** Debug-mode frame times are meaningless; a "fix" validated
  in debug is not validated.

## Step 4 — Web-specific jank

- Layout thrash: reads interleaved with writes in a handler, forcing repeated reflow. Batch reads,
  then writes.
- Scroll handlers not `passive: true`, and layout reads inside them. Prefer `IntersectionObserver`.
- JS-driven animation where CSS would do — CSS runs off the main thread and survives a busy page.
- `will-change` discipline: it costs memory and creates a layer. Apply it just before the animation
  and remove it after; `will-change` on everything is slower than none.
- Motion's `x`/`y`/`scale` shorthands under load — use the full `transform` string.
- A child's transform driven from a CSS variable on the parent, recalculating styles for every child.

---

## Step 5 — Accessibility, added while you are here

If the motion lacks `prefers-reduced-motion` (or Flutter's `MediaQuery.disableAnimations`), add it in
the same pass. Gentler, not zero — keep what aids comprehension, drop translation and scale. Gate
hover motion with `@media (hover: hover) and (pointer: fine)`.

---

## Step 6 — Deleting is a legitimate improvement

**Often the right fix is removal**, especially for motion on a 100+/day action, decoration on
functional data, or the third simultaneous effect on one screen. Say it directly, delete it, and
explain what the interface gains — usually responsiveness.

---

## Output

For each fix: the file and line, the defect named in vocabulary terms, the old value, the new value,
and one line on why. Exact values only — never "make it faster".

Where feel cannot be judged from code — a crossfade, a spring's bounce, a gesture — say so and name
the check: play at 2–5× duration, step frame-by-frame in the animation inspector, test the gesture on
a real device, look again the next day.

## Sources
`frontend-design` (Anthropic); Emil Kowalski's animation philosophy.
