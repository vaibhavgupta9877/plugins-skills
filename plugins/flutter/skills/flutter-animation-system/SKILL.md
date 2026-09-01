---
name: flutter-animation-system
description: Implement Flutter animation mechanics — AnimationController, Tween, curves, Hero, AnimatedBuilder, custom painters, transitions. Pairs with animate/flutter-motion-craft, which decides what should be animated.
---

# flutter-animation-system

> **Boundary.** `animate/flutter-motion-craft` makes the *decision* — what animates, how long, what
> curve, how it should feel, and whether it should animate at all. This skill *implements* that
> decision. **If craft has already specified the motion, implement it exactly — do not re-decide.**
> If no decision exists yet, load `flutter-motion-craft` first rather than inventing timings here.

## Owns
- The API and mechanics of Flutter animation.

## Does not own
- Motion taste, timing, choreography → `animate/flutter-motion-craft`.
- Judging existing motion → `critique/motion-critique`.
- Layout and theming → `flutter-ui-craft`.

---

## 1. Pick the smallest API that works

Most Flutter animation code reaches for an `AnimationController` far too early. Walk down this list
and stop at the first row that fits.

| Need | Use |
|---|---|
| A property changes to a new value | `AnimatedContainer`, `AnimatedOpacity`, `AnimatedAlign`, `AnimatedPadding`, `AnimatedPositioned` |
| Swap one widget for another | `AnimatedSwitcher` |
| Animate on first build, or from an arbitrary value | `TweenAnimationBuilder` |
| Animate a subtree that a parent already drives | `AnimatedBuilder` on the existing `Animation` |
| Orchestration, interruption, reversal, repeat, staggering | `AnimationController` |
| The user drags it, flings it, or it must feel physical | `SpringSimulation` / `AnimationController.animateWith` |
| The widget tree can't express the shape | `CustomPainter` |

Implicit widgets handle interruption for free: change the target mid-flight and they retarget from
the current value. Hand-rolling that with a controller is where most janky Flutter motion comes from.

---

## 2. `AnimationController` lifecycle

```dart
class _FooState extends State<Foo> with SingleTickerProviderStateMixin {
  late final _c = AnimationController(vsync: this, duration: const Duration(milliseconds: 220));
  late final _fade = CurvedAnimation(parent: _c, curve: Curves.easeOutCubic);

  @override
  void dispose() {
    _c.dispose();     // non-negotiable
    super.dispose();
  }
}
```

- `SingleTickerProviderStateMixin` for one controller, `TickerProviderStateMixin` for several.
- **An undisposed controller is a real memory leak and keeps ticking every frame** — the most common
  Flutter animation bug, and it only surfaces after the screen has been opened a few hundred times.
- `forward()` / `reverse()` / `repeat()` / `stop()`; `value` is settable for drag-driven motion.
- Prefer `await _c.forward()` (a `TickerFuture`) over an `AnimationStatus` listener for "when it
  finishes", and guard with `if (!mounted) return;` afterwards.

**Never drive layout from a `Timer` or `setState` in a loop.** `vsync` exists so animations pause
when the widget is off-screen and stay in step with the raster clock.

---

## 3. `AnimatedBuilder` and the `child` parameter

**This is the single highest-impact Flutter animation performance rule.** The builder runs every
frame. Anything static inside it rebuilds 60–120 times a second for no reason.

```dart
AnimatedBuilder(
  animation: _c,
  child: const ExpensiveStaticSubtree(),   // built once
  builder: (context, child) => Opacity(
    opacity: _fade.value,
    child: child,                          // reused every frame
  ),
);
```

Better still, prefer the purpose-built transitions — `FadeTransition`, `SlideTransition`,
`ScaleTransition`, `AlignTransition`, `DecoratedBoxTransition`. They animate at the render-object
level and skip the rebuild entirely. `AnimatedWidget` is the same idea packaged as a reusable class.

---

## 4. Staggering

**One controller, several `Interval`s** — not several controllers.

```dart
Animation<double> _step(int i, int n) => CurvedAnimation(
      parent: _c,
      curve: Interval(i / n * 0.5, i / n * 0.5 + 0.5, curve: Curves.easeOutCubic),
    );
```

Multiple controllers drift, cannot be reversed as a unit, and multiply the disposal surface. With one
controller the sequence has a single `duration`, a single `reverse()`, and one interruption point.
Craft sets the stagger delay (typically 20–50 ms per item, for a handful of items only); convert that
delay into a fraction of the total duration for the `Interval`.

---

## 5. Transitions between screens and widgets

**`PageRouteBuilder`** for custom route transitions — `transitionsBuilder` receives `animation` and
`secondaryAnimation`; using both is what makes the outgoing screen behave correctly when a third
screen pushes on top. With `go_router`, supply a `CustomTransitionPage`.

**`Hero`** for a shared element across routes. Tags must match exactly and be unique per screen. For
anything non-trivial — a different widget shape at each end, a `Material` ancestor problem, text that
would otherwise stretch — supply a `flightShuttleBuilder`. Common failures: duplicate tags in a list
(use the item id), and a `Hero` scrolled off-screen in a `ListView`, which has nothing to fly from.

**`AnimatedSwitcher`** for swapping content in place. **A missing or non-unique `Key` on the child is
why `AnimatedSwitcher` silently does nothing** — it compares by key to decide whether the child
changed. Set `layoutBuilder` when the children differ in size, or the switch jumps.

---

## 6. `CustomPainter`

For progress arcs, charts, signatures, and anything the widget tree can't express.

- Pass the `Animation` as `repaint:` to the painter's constructor — it then repaints without
  rebuilding the widget above it.
- **`shouldRepaint` must compare actual inputs.** `true` unconditionally repaints every frame
  forever; `false` when something changed shows a stale frame. Both are common.
- Wrap in `RepaintBoundary`, and keep `paint()` allocation-free — build `Paint` and `Path` in fields,
  not per frame.

---

## 7. `flutter_animate`

A concise shorthand for chained effects: `.animate().fadeIn(duration: 200.ms).slideY(begin: .04)`.

Genuinely good for simple one-shot entrances. Two costs to state: it hides the controller, so
interruption and reversal get awkward when you later need them; and its fluency makes over-animating
easy — `.shimmer().shake().tint()` costs the same to type as a single fade. Drop to a controller the
moment the motion must be interruptible or gesture-driven.

---

## 8. Physics and gesture-driven motion

When the user is dragging, the motion should continue from their velocity, not restart from a curve.

```dart
final sim = SpringSimulation(
  const SpringDescription(mass: 1, stiffness: 500, damping: 30),
  _c.value, target, velocityFromDrag,
);
_c.animateWith(sim);
```

Set `_c.value` directly from `onPanUpdate`, and hand the velocity to a simulation in `onPanEnd`.
`Curves` are for state changes the user did not physically drive; simulations are for the ones they
did. `flutter-motion-craft` names which case you are in.

---

## 9. Profiling

- **Profile mode on a real device.** Debug-mode frame times include assertion and JIT overhead and
  are meaningless — a debug build that stutters may be perfect in profile.
- DevTools → Performance → enable "Track widget builds". A build phase spike means an
  `AnimatedBuilder` without `child`, or `setState` too high in the tree.
- **UI thread vs raster thread have different fixes.** UI-thread jank = too much Dart work per frame
  (rebuilds, layout, expensive `build`). Raster-thread jank = the GPU is doing too much (opacity
  layers, anti-aliased clips, shadows, `BackdropFilter`, `saveLayer`). Reading the wrong lane sends
  you optimising the wrong thing.
- `Opacity` on a large subtree forces a `saveLayer`; `FadeTransition` is cheaper. `ClipRRect` with
  anti-aliasing on an animating widget is a frequent raster cost. Impeller removes most first-run
  shader jank where it is available.

---

## 10. Respect the system settings

```dart
final reduced = MediaQuery.disableAnimationsOf(context);
final duration = reduced ? Duration.zero : const Duration(milliseconds: 220);
```

`MediaQuery.disableAnimations` reflects the platform accessibility setting — Android's "Remove
animations" and the desktop equivalents. **Honour it globally, not per widget**: derive every
duration in one place so no animation can forget. Android's system animation scale (0.5×, 2×, off)
also multiplies transition durations; test with it off.

A reduced-motion path still communicates the state change — cut to the end state instantly, never
leave the user with no feedback at all.
