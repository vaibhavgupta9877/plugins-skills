---
name: flutter-motion-craft
description: Decide what to animate in a Flutter app and how it should feel — timing, curves, choreography, platform-appropriate motion for Android, Windows, and Linux. Pairs with flutter/flutter-animation-system for the API.
---

# flutter-motion-craft

> **Read first.** Invoke the `frontend-design` skill (plugin `frontend-design@claude-plugins-official`;
> on Linux sandboxes, `/mnt/skills/public/frontend-design/SKILL.md`) and follow its restraint guidance.
> Then load `animation-vocabulary` so the spec you produce is precise.

## Owns
- Flutter motion **taste**: what deserves motion, how long, what it should feel like, per platform.

## Does not own
- The API and controller mechanics → `flutter/flutter-animation-system`.
- Judging existing motion → `critique/motion-critique`.
- Widget styling and layout → `flutter/flutter-ui-craft`.

**The boundary, stated plainly: this skill decides *what and how it feels*;
`flutter/flutter-animation-system` implements it.** Both load when building a Flutter animation —
craft decides, system implements. **If the `flutter` plugin is not enabled, say so and implement
conservatively using implicit animations only.**

---

## Step 1 — The gate applies here too

Default answer is no. Same frequency tiers as `animate`: something the user triggers 100+ times a day
does not animate. A tab switch in a tool used all day is not a Hero transition candidate.

Name the purpose — feedback · spatial consistency · state indication · preventing a jarring change ·
delight (rare tier only) — or don't build it.

---

## Step 2 — Prefer implicit animations

**Most Flutter animation code reaches for an `AnimationController` where an implicit widget would
do.** A controller means a `TickerProvider`, a `dispose()`, and lifecycle bugs — pay that cost only
when you need it.

Reach for implicit widgets first: `AnimatedContainer`, `AnimatedOpacity`, `AnimatedAlign`,
`AnimatedPadding`, `AnimatedSwitcher`, `AnimatedSize`, `TweenAnimationBuilder`.

Escalate to an explicit `AnimationController` only for **orchestration** (several things on one
timeline), **interruption control**, **reversal**, or a **driven sequence**. Escalate to a
`SpringSimulation` when the user is dragging it.

Hand the implementation to `flutter/flutter-animation-system` once you have decided.

---

## Step 3 — Curve defaults

| Situation | Curve |
|---|---|
| Entrance | `Curves.easeOutCubic` |
| Exit | `Curves.easeInCubic` (and shorter than the entrance) |
| Moving while present | `Curves.easeInOutCubic` |
| Gesture-driven | `SpringSimulation`, not a curve |
| Continuous / looping | `Curves.linear` |

**Do not use `Curves.easeInOut` reflexively.** It is the Flutter equivalent of leaving everything on
the default, and it makes entrances feel hesitant. The `*Cubic` variants are stronger and read as
deliberate.

Durations: 100–150ms micro-feedback, 200–300ms state change, 300–400ms route transition. Exits
0.6–0.8× the entrance. Anything past 500ms needs a written reason.

---

## Step 4 — Platform-appropriate motion is a constraint, not a nicety

**A single motion spec across Android, Windows, and Linux will feel wrong on at least one of them.**
Branch on `defaultTargetPlatform` and on form factor.

**Android** expects Material motion: shared axis for peer-to-peer navigation, fade-through for
unrelated destinations, container transform for a card opening into a detail view. It also follows
the **system animation scale** — a user who set animations to 0.5× expects your app to obey.

**Windows and Linux desktop** expect *less*:
- **~30% shorter durations.** Pointer precision and instant clicks make mobile timings feel sluggish.
- **No bouncy springs.** Bounce reads as toy-like on desktop; keep it near-critically damped.
- **Hover is now a real state** and must be designed — it does not exist on mobile.
- **Window resize must never animate.** Layout follows the drag 1:1, or it feels broken.
- Scroll physics differ: a wheel delivers discrete deltas, not momentum. Do not port a fling
  animation to a wheel.
- Desktop users see the same screen for hours. Motion that charms once a day irritates at 09:15.

Write the spec as a small table — surface · Android · desktop — rather than one duration everyone has
to reinterpret.

---

## Step 5 — Choreography

One orchestrated moment per screen. Scattered independent animations are noise.

- Stagger a list at 30–80ms per item via `Interval` on **one** shared controller, never several
  controllers. Cap the total; do not stagger a long list.
- **Hero transitions: worth it for one primary flow, noise everywhere else.** A Hero on every list in
  the app makes navigation feel heavy and makes the one meaningful transition unremarkable.
- Do not run a route transition and a content entrance simultaneously — one or the other.

---

## Step 6 — Accessibility

**Honour `MediaQuery.of(context).disableAnimations`.** Reduce to opacity or instant — never disable
in a way that leaves the widget in a broken intermediate state.

Read it once near the top of the tree and thread it through your motion spec, rather than checking it
in twenty widgets.

---

## Step 7 — Performance rules that actually matter

State these in the spec, because they change what is buildable:

- **Use `AnimatedBuilder`'s `child` parameter** to keep static subtrees out of the per-frame rebuild.
  This is the highest-impact Flutter animation performance rule.
- **`RepaintBoundary` around an animating region**, so neighbours don't repaint with it.
- **Prefer `FadeTransition` / `AnimatedOpacity` over `Opacity`** — the latter forces a `saveLayer`.
- No allocation, list-building, or `MediaQuery` lookup inside `build()` during an animation.
- **Profile in profile or release mode only.** Debug-mode jank is meaningless, and a spec validated
  in debug is not validated.

---

## Output

A short motion spec, then hand off:

| Surface | Trigger | Purpose | Android | Desktop | API |
|---|---|---|---|---|---|
| Detail open | Tap card | Spatial consistency | Container transform, 300ms `easeOutCubic` | 220ms, no bounce | `flutter/flutter-animation-system` |

Then the restraint check: if the spec has more than one animated moment per screen, justify each or
cut it. When the honest answer is that a screen needs no motion, say so.

## Sources
`frontend-design` (Anthropic); Material motion guidance; Emil Kowalski's animation philosophy.
