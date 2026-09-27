# Plugin: `animate`

**Enable for:** any project with a UI (Next.js, company site, Flutter).
**Purpose:** motion craft. Everything about *making* animation good.
**Skills:** 7

**Boundary, stated once and repeated in every body:**
- `animate` **makes** motion. `critique/motion-critique` **judges** it. This plugin contains no review skill.
- `flutter/flutter-animation-system` is the **API**: `AnimationController`, `Tween`, curves, `Hero`, `CustomPainter`, ticker lifecycle. `animate/flutter-motion-craft` is the **taste**: what deserves motion, how long, how it should feel. When building a Flutter animation, both load — craft decides, system implements.

**Every skill in this plugin opens with the R4 `frontend-design` directive.** Its restraint guidance ("extra animation contributes to the feeling that the design is AI-generated") is the counterweight to a plugin whose entire purpose is adding animation. Do not let this plugin become an animation-maximiser.

`product-direction/experience-direction` may specify a narrative motion purpose. When `docs/product-direction.md` has a matching surface section, `scroll-motion` reads its constraints; if the section is absent, it uses the current task and its frequency gate, with no cinematic motion by default. It still owns motion implementation and quality gates.

**Sources for the whole plugin (fetch and distil per R7, do not vendor):**
- `github.com/emilkowalski/skills/tree/main/skills` — `animate`, `animate-vocabulary`, `emil-design-eng`, `find-animation-opportunities`, `improve-animations`
- `github.com/ConardLi/garden-skills/tree/main/skills/web-design-engineer`
- `github.com/MengTo/Skills/tree/main/agent-skills/web-design`
- `github.com/jakubkrehel/skills/tree/main/skills` — `better-ui`
- `github.com/codeswithroh/tastemaker`
- The user also referenced `animation-on-scroll`, `cinematic-scroll-storytelling`, `build-interactive-particle-trail`, `build-awwwards-quality-sites`. **Search these by name across the repos above and any others; treat as concept sources.** `build-awwwards-quality-sites` is landing-page scope → its findings go to `web-ui/landing-page-craft`, not here. `build-interactive-particle-trail` is too narrow to justify a skill — fold any useful heuristics into `animate` and do not create a separate skill for it.

---

## `animate` — CORE

**description:** `Design and implement motion for a UI element or interaction — timing, easing, springs, transforms. The default entry point for adding animation. Do NOT use to review existing motion — that's critique/motion-critique.`

**Owns:** core motion craft, timing and easing decisions, implementation in CSS/Motion/Flutter.
**Does NOT own:** terminology → `animation-vocabulary`; finding where to animate → `find-animation-opportunities`; fixing existing motion → `improve-animations`; scroll-linked → `scroll-motion`.

**Body must cover:**
- **[R4] Read `/mnt/skills/public/frontend-design/SKILL.md` first.**
- Default answer to "should this animate" is **no**. Motion is justified when it explains a spatial or causal relationship, maintains continuity across a state change, or gives feedback that the system heard the user. Decoration is not a justification.
- Duration defaults that are correct far more often than the model's instinct: micro-feedback (hover, press) 100–150ms · state change / reveal 200–300ms · layout or route transition 300–400ms · anything past 500ms needs a written reason. Exits are faster than entrances, always.
- Easing: never linear except for continuous/looping motion. Entering elements decelerate (`ease-out`). Exiting elements accelerate (`ease-in`). `ease-in-out` for elements that move while staying present. Springs for anything the user directly manipulates — mass/stiffness/damping beats duration for gestural motion.
- Animate only compositor-safe properties on web: `transform` and `opacity`. Animating `width`, `height`, `top`, `left`, `margin`, or `box-shadow` is a defect. Use `FLIP` for layout transitions.
- Interruptibility is non-negotiable. An animation the user can't interrupt is a bug. Motion must be re-targetable mid-flight.
- Honour `prefers-reduced-motion` in every implementation — reduce to opacity-only or instant, never just disable and leave a broken state.
- Stack implementations: web → CSS transitions first, Motion (Framer Motion) only when state orchestration demands it; Flutter → hand off to `flutter-motion-craft`.
- Close with the restraint check: if you've added more than one animated moment to a screen, justify each or cut it.

**reference/:** `timing-table.md`, `spring-presets.md`.

**Budget:** 200 lines.

---

## `animation-vocabulary` — SUPPORTING

**description:** `Shared precise terminology for describing motion — easing curves, springs, stagger, orchestration, choreography. Load when discussing or specifying animation so descriptions are exact rather than vague.`

**Owns:** the shared language. Referenced by `animate`, `improve-animations`, `flutter-motion-craft`, and `critique/motion-critique`.
**Does NOT own:** any implementation.

**Body must cover:**
- Definitions with one-line examples: easing families and what each communicates · spring parameters (stiffness, damping, mass) and what changing each *feels* like · stagger and its delay ranges · orchestration vs choreography · enter/exit/layout transition classes · interruption vs queuing vs re-targeting · scrub vs trigger for scroll · anticipation, overshoot, settle.
- Include the web term ↔ Flutter term mapping table (`ease-out` ↔ `Curves.easeOut`, spring ↔ `SpringSimulation`, stagger ↔ `Interval` on a shared controller, layout transition ↔ `AnimatedSwitcher`/`Hero`).
- This is a **reference skill**. Pure definitions, no procedure. Keep it dense and short.

**Budget:** 90 lines.

---

## `find-animation-opportunities` — SUPPORTING

**description:** `Scan an existing static UI and identify where motion would add real value, ranked by payoff. Do NOT use to implement the motion — that's animate.`

**Owns:** opportunity discovery and ranking.
**Does NOT own:** implementation → `animate`; improving existing motion → `improve-animations`.

**Body must cover:**
- **[R4] frontend-design directive.**
- Scan for the four high-payoff categories only: state changes with no visual continuity (content swaps that jump) · actions with no feedback (a press that appears to do nothing) · spatial relationships that are asserted but not shown (a panel that appears with no origin) · loading and empty states that read as broken.
- Rank by payoff, and **state how many to actually implement** — usually 2–3 per screen, not everything found. A list of twelve opportunities is a failure of judgement, not thoroughness.
- Explicitly output an "opportunities deliberately not taken" section. This is what stops the plugin from over-animating.

**Budget:** 90 lines.

---

## `improve-animations` — SUPPORTING

**description:** `Fix motion that already exists but feels wrong — sluggish, janky, distracting, or uninterruptible. Do NOT use to judge quality without changing it — that's critique/motion-critique.`

**Owns:** refining existing motion.
**Does NOT own:** judgement-only output → `critique/motion-critique`; new motion → `animate`.

**Body must cover:**
- Diagnose before changing. Load `animation-vocabulary` and name the actual defect.
- The five defects that account for most bad motion, with fixes: too slow (halve it) · wrong easing direction on exit · animating non-compositor properties (jank) · no interruption handling (feels stuck) · too many simultaneous independent animations (noise — consolidate into one orchestrated sequence or cut).
- Flutter-specific jank causes: animating above a rebuild-heavy subtree instead of using `AnimatedBuilder`'s `child` parameter, missing `RepaintBoundary`, opacity animations forcing saveLayer, work in `build()` during animation.
- Web-specific: layout thrash, missing `will-change` discipline, scroll handlers not passive, JS-driven animation where CSS would do.
- Deleting an animation is a legitimate improvement and often the right one.

**Budget:** 120 lines.

---

## `scroll-motion` — SUPPORTING

**description:** `Build scroll-triggered reveals, scroll-scrubbed sequences, parallax, and cinematic scroll storytelling on the web. Do NOT use for non-scroll animation — that's animate.`

**Owns:** everything scroll-linked on web — reveals, scrub, pinning, parallax, narrative sequences.
**Does NOT own:** general motion → `animate`; Flutter scroll effects → `flutter-motion-craft`.

**Body must cover:**
- **[R4] frontend-design directive.**
- Two distinct modes, chosen deliberately: **triggered** (fires once when an element enters view — use `IntersectionObserver`, cheap, safe, correct for 90% of cases) and **scrubbed** (progress tied to scroll position — expensive, powerful, only for a deliberate narrative moment).
- Never hijack scroll. Never disable native scrolling. Momentum, trackpad, wheel, touch, and keyboard must all behave. Test all four.
- Reveals: subtle and fast. 200–300ms, small translate (8–24px), opacity. Large slide-ins and 600ms fades are the standard AI tell.
- Scrubbed sequences: pin the section, map scroll progress to a normalised 0–1 timeline, drive transforms from that. Budget one such sequence per page — this is the "spend your boldness in one place" rule applied to scroll.
- Performance: `IntersectionObserver` over scroll listeners; if a scroll listener is unavoidable, `passive: true` + rAF throttle. Never read layout in a scroll handler.
- Must degrade: content is fully readable with JS disabled and under `prefers-reduced-motion`. Content that only exists after a scroll trigger is a content bug.
- Mobile: scrubbed scroll behaves differently on iOS momentum. Either simplify to triggered on small viewports or test on device.

**Sources:** the `animation-on-scroll` and `cinematic-scroll-storytelling` concepts named by the user.

**Budget:** 160 lines.

---

## `design-engineering` — CORE

**description:** `Build interaction detail that makes an interface feel hand-crafted — hover, press, focus, drag, optimistic states, loading, transitions between states. Use when polishing UI feel. Do NOT use for layout or visual design — that's web-ui/*.`

**Owns:** the craft layer between design and engineering — interaction states, perceived performance, micro-detail.
**Does NOT own:** visual/layout design → `web-ui/design-system`, `web-ui/web-design-engineering`; pure animation timing → `animate`.

**Body must cover:**
- **[R4] frontend-design directive.**
- Every interactive element has five states, and most implementations ship two: default, hover, **active/press**, **focus-visible**, **disabled**. Plus loading and error for anything async. The missing press state is the most common reason a UI feels cheap.
- Perceived performance beats measured performance: optimistic updates, skeleton states matched to the real layout (not generic grey bars), instant feedback on press before the network resolves, and never a spinner where a skeleton or an optimistic result would do.
- Async state discipline: idle → pending → success → error, all four designed. Errors are recoverable in place, not a page replacement.
- Detail that signals craft: correct cursor per affordance, text selection disabled only on controls, focus rings that are visible and not ugly, hit areas larger than visual bounds, tabular numerals for changing numbers, no layout shift when content loads.
- Drag/gesture: follow the pointer 1:1, resist at boundaries, spring on release, cancel cleanly on escape.
- This skill is the highest-leverage one in the plugin for "doesn't look AI-generated." Weight it accordingly.

**Sources:** `emil-design-eng`, `better-ui`, `tastemaker`, `web-design-engineer`, MengTo `web-design`.

**Budget:** 200 lines.

---

## `flutter-motion-craft` — CORE

**description:** `Decide what to animate in a Flutter app and how it should feel — timing, curves, choreography, platform-appropriate motion for Android, Windows, and Linux. Pairs with flutter/flutter-animation-system for the API.`

**Owns:** Flutter motion taste and decisions.
**Does NOT own:** the animation API and controller mechanics → `flutter/flutter-animation-system`; judging existing motion → `critique/motion-critique`.

**Body must cover:**
- **[R4] frontend-design directive.** Then load `animation-vocabulary`.
- **State the boundary explicitly:** this skill decides *what and how it feels*; `flutter/flutter-animation-system` implements it. If that plugin isn't enabled, say so and implement conservatively with implicit animations only.
- Prefer implicit animations (`AnimatedContainer`, `AnimatedOpacity`, `AnimatedSwitcher`, `TweenAnimationBuilder`) for state changes. Reach for an explicit `AnimationController` only when you need orchestration, interruption control, or a driven sequence. Most Flutter animation code uses a controller where an implicit widget would do.
- Curve defaults: `Curves.easeOutCubic` for entrances, `Curves.easeInCubic` for exits, `Curves.easeInOutCubic` for moves, spring simulations for gesture-driven motion. Do not use `Curves.easeInOut` reflexively.
- **Platform-appropriate motion is a real constraint, not a nicety.** Android expects Material motion (shared axis, fade-through, container transform) and follows system animation scale. Windows and Linux desktop expect *less* motion, faster durations, no bouncy springs, and hover states that mobile doesn't have. A single motion spec across all three platforms will feel wrong on at least one. Branch on `defaultTargetPlatform` / form factor.
- Desktop-specific: window resize must not animate, hover is now a real state, pointer precision means smaller hit targets are acceptable but motion should be snappier (~30% shorter durations than mobile).
- Honour `MediaQuery.of(context).disableAnimations`.
- Performance rules that matter: use `AnimatedBuilder`'s `child` to avoid rebuilding static subtrees, add `RepaintBoundary` around animating regions, avoid `Opacity` in favour of `FadeTransition`/`AnimatedOpacity`, profile in **release/profile mode only** — debug-mode jank is meaningless.
- Hero transitions across routes: worth it for one primary flow, noise everywhere else.

**Budget:** 190 lines.
