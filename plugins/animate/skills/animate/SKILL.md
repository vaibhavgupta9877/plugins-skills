---
name: animate
description: Design and implement motion for a UI element or interaction — timing, easing, springs, transforms. The default entry point for adding animation. Do NOT use to review existing motion — that's critique/motion-critique.
---

# animate

> **Read first.** Invoke the `frontend-design` skill (plugin `frontend-design@claude-plugins-official`;
> on Linux sandboxes, `/mnt/skills/public/frontend-design/SKILL.md`) and follow it. It is the source of
> truth for aesthetic direction and restraint. **Its warning that extra animation makes a design feel
> AI-generated is the counterweight to this entire plugin.** This skill never contradicts it.

## Owns
- Core motion craft: the decision to animate, timing, easing, springs, transforms.
- Implementation in CSS, WAAPI, and Motion.

## Does not own
- Terminology → `animation-vocabulary`.
- Finding where motion is missing → `find-animation-opportunities`.
- Fixing motion that already exists → `improve-animations`.
- Scroll-linked motion → `scroll-motion`.
- Flutter motion → `flutter-motion-craft` (taste) + `flutter/flutter-animation-system` (API).
- Judging motion → `critique/motion-critique`.

Run the sequence **in order**. Steps 1 and 2 gate everything: do not reach for a curve before you
know whether it animates at all.

---

## Step 1 — Should this animate at all?

**The default answer is no.** This gate exists to produce zero lines of code sometimes; that is a
success, not a dodge.

| How often the user sees it | Decision |
|---|---|
| 100+/day — keyboard shortcuts, command palette, core nav | **No animation. Ever.** Stop here. |
| Tens/day — hover, list navigation, frequent toggles | Near-imperceptible only, or nothing |
| Occasional — modals, drawers, toasts | Standard animation |
| Rare / first-time — onboarding, success, celebration | The delight budget lives here |

**Keyboard-initiated actions are a disqualifier, not a judgement call.** Raycast has no open/close
animation, and that is the correct design for something opened hundreds of times a day.

Also check **function**: data the user is reading or acting on must not move for style. A decorative
mouse-tracking effect belongs on a marketing page, not on a chart in a banking app.

If the request fails the gate, say so plainly, don't write it, and offer the non-motion alternative —
an instant state change, or a static affordance.

---

## Step 2 — Name the purpose

One of these words, before continuing: **feedback** (confirming the interface heard the user) ·
**spatial consistency** (showing where something came from or went) · **state indication** (making a
change legible) · **preventing a jarring change** (bridging content that would teleport) ·
**explanation** (marketing and onboarding only) · **delight** (rare/first-time tier only).

Can't name it? Don't build it. "It looks cool" on a frequently-seen element is a reason to stop.

---

## Step 3 — Pick the cheapest tool that works

Walk down; stop at the first that fits. Don't install a motion library for a fade.

| Need | Tool |
|---|---|
| Hover, press, colour, a state toggle you control with a class or attribute | **CSS transition** |
| Entry animation on mount, no JS state | **CSS `@starting-style`** |
| Predetermined motion that must stay smooth while the page is busy | **CSS animation** (runs off the main thread) |
| Programmatic control with CSS performance, no library | **WAAPI** (`element.animate()`) |
| Springs, layout animation, exit animation, gesture-driven values | **Motion** (`motion.dev`) |

CSS beats JS under load — `requestAnimationFrame` motion drops frames while the browser is loading,
scripting, or painting. CSS for predetermined motion, JS for dynamic and interruptible motion.

**Extend the codebase's tokens, don't fork them.** If `--ease-out` or a duration scale already
exists, use it. A parallel system is a defect.

---

## Step 4 — Pick the properties

- **`transform` and `opacity` only.** They skip layout and paint and run on the GPU. `width`,
  `height`, `margin`, `padding`, `top`, `left` trigger all three. `clip-path` is the sanctioned
  fourth; `height` is tolerated only for accordions, which have no transform equivalent.
- **Never `scale(0)`.** Start from `scale(0.9–0.97)` with `opacity: 0`. Nothing in the real world
  appears from nothing.
- **`transform-origin` at the trigger** for popovers, dropdowns, menus, tooltips —
  `var(--transform-origin)` in Base UI. **Modals are exempt**: not anchored to a trigger, so centre.
- **Percentages in `translate()`** are relative to the element's own size — `translateY(100%)` moves
  by its own height whatever the content is. Prefer over hardcoded pixels.
- **In Motion, animate the full transform string.** `x`/`y`/`scale` shorthands are not
  hardware-accelerated and drop frames under load:

```jsx
<motion.div animate={{ x: 100 }} />                          // drops frames under load
<motion.div animate={{ transform: "translateX(100px)" }} />  // hardware accelerated
```

- Never drive a child's transform from a CSS variable on the parent — it recalculates styles for
  every child. Set `transform` on the element directly.
- For a layout change you cannot express as a transform, use **FLIP**: measure first, measure last,
  invert with a transform, play. Never animate the layout properties themselves.

---

## Step 5 — Easing and duration, or a spring

**Easing**, in decision order:

| Situation | Easing |
|---|---|
| Entering or exiting | `ease-out` |
| Moving / morphing while on screen | `ease-in-out` |
| Hover / colour change | `ease` |
| Constant motion (marquee, progress) | `linear` |
| Default | `ease-out` |

**Never `ease-in` on UI.** It starts slow, delaying exactly the moment the user is watching.
**Never `linear` except for continuous or looping motion.**

Built-in CSS easings are too weak for deliberate motion. Use these:

```css
--ease-out:    cubic-bezier(0.23, 1, 0.32, 1);   /* strong ease-out for UI */
--ease-in-out: cubic-bezier(0.77, 0, 0.175, 1);  /* on-screen movement */
--ease-drawer: cubic-bezier(0.32, 0.72, 0, 1);   /* iOS-like drawer curve */
```

**Duration** — see `reference/timing-table.md` for the full table and the reasoning:

micro-feedback (hover, press) **100–150ms** · state change / reveal **200–300ms** · layout or route
transition **300–400ms**. Anything past 500ms needs a written reason. **Exits are faster than
entrances, always** — the user has already decided and is now waiting.

**Reach for a spring instead** when the motion is drag with momentum, something that should feel
alive, a gesture the user can interrupt or reverse, or decorative pointer-tracking. Mass, stiffness
and damping beat duration for anything the user directly manipulates. Presets and what changing each
parameter *feels* like: `reference/spring-presets.md`.

---

## Step 6 — Interruption and exit

**Interruptibility is non-negotiable. An animation the user cannot interrupt is a bug.**

- **Transitions, not keyframes, for anything triggered rapidly** — toasts, toggles, anything firable
  twice a second. Transitions re-target from the current value; keyframes restart from zero.
- **Springs for gestures**, because they carry velocity through an interruption.
- **Exit the way it entered.** A toast that slides in from the bottom leaves through the bottom.
  Symmetric paths are what make swipe-to-dismiss feel obvious.
- **Asymmetric timing where the user is deciding**: slow on the deliberate phase (hold-to-confirm:
  2s linear), snappy on the system response (release: 200ms ease-out).

---

## Step 7 — Reduced motion and pointer gating

Ships with the animation, every time. Not a follow-up.

```css
@media (prefers-reduced-motion: reduce) {
  .element { transition: opacity 200ms ease; transform: none; }
}

@media (hover: hover) and (pointer: fine) {
  .element:hover { transform: scale(1.02); }  /* touch fires false hovers on tap */
}
```

**Reduced motion means fewer and gentler, not zero.** Keep transitions that aid comprehension, drop
translation and scale. Never disable an animation in a way that leaves the element in a broken
intermediate state.

---

## Step 8 — Restraint check, then output

**If you have added more than one animated moment to a screen, justify each or cut it.** An
orchestrated moment lands harder than scattered effects. Before you finish, remove one accessory.

Write the code. Then, in a few lines at most:

- **Gate result** — frequency tier and the named purpose. If you rejected part of the request, say
  which and why.
- **Ingredients** — tool, properties, curve, duration or spring config. One line each.
- **What to feel-check** — where the result depends on feel you cannot judge from code (a crossfade,
  a spring's bounce, an entering list), say so and name the check: play at 2–5× duration, step
  frame-by-frame in the animation inspector, test gestures on a real device.

Never present motion options as a menu. Make the call, state the reasoning in one line, write the
code. When the honest answer is "this shouldn't animate," give it.

## Sources
`frontend-design` (Anthropic); Emil Kowalski's animation philosophy.
