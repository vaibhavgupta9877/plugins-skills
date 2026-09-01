# Spring presets

A spring has no duration — it settles according to physics. That is the point: it carries velocity
through an interruption, so a gesture reversed mid-flight looks continuous rather than restarted.

**Use a spring when** the motion is drag with momentum, an element that should feel alive under
direct manipulation, a gesture the user can interrupt or reverse, or decorative pointer-tracking.

**Use a duration + curve when** the motion is a discrete state change the user is not manipulating —
a dropdown opening, a toast arriving, a route transition. Springs there are unpredictable and usually
slower than they look.

---

## Two ways to specify (Motion)

```js
// Apple-style — easier to reason about, prefer this
{ type: "spring", duration: 0.5, bounce: 0.2 }

// Traditional physics — more control
{ type: "spring", mass: 1, stiffness: 100, damping: 10 }
```

`duration` + `bounce` is the better default: you say how long it should feel and how much overshoot
you want, and the solver picks the physics.

---

## What each parameter feels like

| Parameter | Raise it | Lower it |
|---|---|---|
| **stiffness** | Snappier, arrives sooner, more aggressive | Lazier, floatier, slower to commit |
| **damping** | Settles quickly, less oscillation; too high feels dead | Bouncier, wobbles longer; too low never settles |
| **mass** | Heavier, more inertia, slower to start and stop | Lighter, more darting, more nervous |
| **bounce** | More overshoot past the target | 0 = no overshoot, critically damped |

Tune **one** at a time. Changing stiffness and damping together is how you end up with a spring you
cannot reason about.

---

## Presets

| Use | Config | Feel |
|---|---|---|
| Default UI spring | `{ duration: 0.5, bounce: 0.2 }` | Lively but controlled |
| Snappy / responsive control | `{ duration: 0.3, bounce: 0.1 }` | Quick, barely overshoots |
| No-overshoot (functional) | `{ duration: 0.4, bounce: 0 }` | Spring feel, lands exactly |
| Drag release | `{ duration: 0.5, bounce: 0.25 }` | Carries velocity, settles naturally |
| Playful / delight tier only | `{ duration: 0.6, bounce: 0.35 }` | Visible bounce |
| Heavy sheet or drawer | `{ mass: 1.2, stiffness: 200, damping: 26 }` | Weighted, deliberate |
| Pointer tracking | `{ mass: 0.5, stiffness: 150, damping: 15 }` | Trails the cursor with lag |

**Keep bounce at 0.1–0.3 for functional UI.** Above ~0.3 reads as toy-like and belongs only in the
rare/first-time delight tier. A bouncy spring on a settings toggle is a finding, not a flourish.

---

## Gesture rules

- **Follow the pointer 1:1** while dragging. Any smoothing during the drag itself feels like lag.
- **Resist at boundaries** — rubber-band with diminishing returns rather than stopping hard.
- **Dismiss on velocity, not only distance.** A fast short flick should dismiss; a slow long drag
  that stops short should snap back. `Math.abs(distance) / elapsedMs > ~0.11` is a reasonable
  threshold to start from.
- **Spring on release**, seeded with the release velocity.
- **Cancel cleanly** on Escape or pointer-cancel, returning to the origin with the same spring.

---

## Flutter equivalent

`SpringSimulation` driven through an `AnimationController`:

```dart
final spring = SpringDescription(mass: 1, stiffness: 100, damping: 10);
controller.animateWith(SpringSimulation(spring, from, to, velocity));
```

`damping` here is the raw damping coefficient, not a ratio — critical damping for `mass: 1,
stiffness: 100` is `2 * sqrt(100 * 1) = 20`. Below that oscillates, above is sluggish.

Use a spring for anything the user drags (sheets, dismissible cards, sliders); use a curve for
discrete state changes. See `flutter-motion-craft` for the decision and
`flutter/flutter-animation-system` for the mechanics.

---

## Reduced motion

A spring is still motion. Under `prefers-reduced-motion` (or `MediaQuery.disableAnimations`), replace
it with an opacity transition or an instant change — do not merely stiffen it.
