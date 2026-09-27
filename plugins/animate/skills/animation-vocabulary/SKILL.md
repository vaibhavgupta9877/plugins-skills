---
name: animation-vocabulary
description: Use when precise terminology is needed to describe motion across web and Flutter. Do NOT use to choose or implement motion — that's `animate`, `animate/flutter-motion-craft`, or `flutter/flutter-animation-system`.
---

# animation-vocabulary

> Before doing anything else, read `/mnt/skills/public/frontend-design/SKILL.md` and follow it. It is the source of truth for aesthetic direction, typography, palette, restraint, and interface copy. This skill adds stack-specific execution on top of it and never contradicts it.

## Owns
- The shared language for describing motion. Definitions only.

## Does not own
- Any implementation or decision. Referenced by `animate`, `improve-animations`,
  `flutter-motion-craft`, and `critique/motion-critique`.

**This is a reference skill.** Use it so findings and specs say "`ease-in` on an exit at 450ms"
instead of "feels off."

---

## Easing families

| Term | Definition | Communicates |
|---|---|---|
| **linear** | Constant rate | Mechanical. Correct only for continuous/looping motion |
| **ease-out** | Fast start, slow settle | Arrival. The default for UI — response is visible immediately |
| **ease-in** | Slow start, fast end | Departure. Never on UI, since it delays the watched moment |
| **ease-in-out** | Slow both ends | An element moving while staying present |
| **cubic-bezier(x1,y1,x2,y2)** | Two control points on a 0–1 curve | The precise form of all of the above |

Named curves used across this plugin:
`--ease-out: cubic-bezier(0.23, 1, 0.32, 1)` · `--ease-in-out: cubic-bezier(0.77, 0, 0.175, 1)` ·
`--ease-drawer: cubic-bezier(0.32, 0.72, 0, 1)`.

---

## Springs

| Term | Definition | Raising it feels like |
|---|---|---|
| **stiffness** | Restoring force toward the target | Snappier, arrives sooner |
| **damping** | Resistance that removes energy | Settles faster; too high feels dead |
| **mass** | Inertia | Heavier, slower to start and stop |
| **bounce** | Normalised overshoot (0 = none) | More overshoot past the target |
| **critically damped** | Reaches target fastest with no overshoot | The functional default |
| **underdamped** | Oscillates before settling | Playful; delight tier only |

---

## Timing structure

- **Duration** — fixed time for a curve-based animation. Springs have no duration; they settle.
- **Delay** — wait before starting. Reads as lag on functional UI; see the tooltip exception.
- **Stagger** — a fixed delay increment applied across a group. 30–80ms per item; cap the total
  around 300ms.
- **Orchestration** — the *mechanism* that coordinates several animations: one shared timeline or
  controller, sequenced offsets.
- **Choreography** — the *design* of how those animations relate: what leads, what follows, what the
  sequence means. Orchestration is plumbing; choreography is intent.
- **Anticipation** — a small movement opposite the main direction before it starts. Rare in UI.
- **Overshoot** — passing the target before returning. A spring property, or a curve with y > 1.
- **Settle** — the final approach to rest.

---

## Transition classes

- **Enter** — element appears. `ease-out`, from `scale(0.95)` + `opacity: 0`, never `scale(0)`.
- **Exit** — element leaves. Same path as its entrance, reversed, at ~0.6–0.8× the duration.
- **Layout transition** — element stays but changes position or size. FLIP on web;
  `AnimatedSwitcher`/`Hero` in Flutter.
- **Shared element** — the same content animating between two screens, preserving identity.
- **Crossfade** — one element fades out while another fades in. Blur during the swap masks a
  mismatch between the two.
- **Morph** — one shape becomes another rather than being replaced.

---

## Interruption

- **Queuing** — the new animation waits for the current one. Almost always wrong; feels stuck.
- **Restarting** — the new animation begins from its initial value. What keyframes do, and why they
  jump on rapid triggers.
- **Re-targeting** — the animation changes destination from its *current* value and velocity. What
  CSS transitions and springs do. This is what "interruptible" means.
- **Cancel-safe** — the motion can be abandoned mid-flight without leaving a broken visual state.

---

## Scroll

- **Triggered** — fires once when an element enters the viewport. `IntersectionObserver`. Cheap,
  safe, correct for ~90% of cases.
- **Scrubbed** — progress is bound to scroll position; scrolling back reverses it. Expensive.
- **Pinning** — the section holds still while the scroll drives an internal timeline.
- **Parallax** — layers moving at different rates to imply depth.
- **Progress (0–1)** — the normalised scrub value everything else is derived from.

---

## Performance

- **Compositor-only** — `transform` and `opacity`, which skip layout and paint. Everything else janks.
- **Layout thrash** — interleaved reads and writes forcing repeated reflow, usually in a handler.
- **Repaint boundary** — an isolated region so animating it doesn't repaint its neighbours.
  `RepaintBoundary` in Flutter; `will-change` used sparingly on web.
- **Raster vs UI thread** (Flutter) — different symptoms, different fixes: raster jank is painting
  cost, UI jank is `build()` cost.

---

## Web ↔ Flutter mapping

| Web | Flutter |
|---|---|
| `ease-out` | `Curves.easeOutCubic` |
| `ease-in-out` | `Curves.easeInOutCubic` |
| `linear` | `Curves.linear` |
| spring | `SpringSimulation` + `SpringDescription` |
| `transition` on a property | `AnimatedContainer`, `AnimatedOpacity`, `AnimatedAlign` |
| keyframes / timeline | `AnimationController` + `TweenSequence` |
| stagger | `Interval` on one shared controller |
| layout transition | `AnimatedSwitcher` (with keys), `AnimatedSize` |
| shared element | `Hero` (+ `flightShuttleBuilder`) |
| `IntersectionObserver` | `VisibilityDetector` |
| `prefers-reduced-motion` | `MediaQuery.of(context).disableAnimations` |
| `will-change` / compositor layer | `RepaintBoundary` |
| Motion (`motion.dev`) | `flutter_animate` (shorthand) or a controller |
