# Plugin: `flutter`

**Enable for:** Flutter projects targeting Android, Windows, and Linux.
**Purpose:** Flutter application engineering. Motion *taste* lives in `animate/flutter-motion-craft`; the motion *API* lives here.
**Skills:** 5

**Boundary reminder for the agent:** `flutter/flutter-animation-system` = the API and mechanics. `animate/flutter-motion-craft` = what to animate and how it should feel. Both load when building an animation; craft decides, system implements. Neither judges — that's `critique/motion-critique`.

**Convention detection is mandatory.** This codebase's state management, routing, and DI choices must be read from the existing project (`pubspec.yaml`, `lib/` structure) before any skill imposes a pattern. If the project is new, `core-eng/grill-me` asks. Never silently pick Riverpod over Bloc.

When `docs/product-direction.md` exists, `flutter-ui-craft` consumes Brand foundation and any matching surface section that is present. If the surface section is absent, it uses the current brief and existing Flutter UI; the missing section does not block routine implementation. It translates available direction through Flutter's adaptive/platform-specific rules. `product-direction` remains cross-stack and does not implement Flutter UI.

---

## `flutter-architecture` — CORE

**description:** `Structure a Flutter application — layering, state management, dependency injection, routing, data and repository layers, offline behaviour. Do NOT use for widget-level UI styling, that's flutter-ui-craft.`

**Owns:** project structure, layering, state management application, DI, routing, repositories, local persistence, offline/sync strategy.
**Does NOT own:** visual/widget design → `flutter-ui-craft`; animation → `flutter-animation-system`; desktop platform specifics → `flutter-desktop-platform`; planning documents → `core-eng/spec-writer`.

**Body must cover:**
- **Detect first.** Read `pubspec.yaml` and `lib/` before proposing anything. Follow the project's existing state management. Only when starting fresh does this skill choose, and then it states the tradeoff and asks.
- Layering: `presentation` (widgets, controllers) → `domain` (entities, use-cases, repository interfaces) → `data` (repository implementations, data sources, DTOs). Dependencies point inward only. `BuildContext` never crosses out of `presentation`. Flag every violation.
- Feature-first directory layout (`lib/features/<feature>/{presentation,domain,data}`) with `lib/core/` for genuinely shared code. Layer-first layouts stop scaling around feature five.
- State management guidance without dogma: name what each is actually good at (Riverpod — compile-safe DI plus reactive state, best default for new work; Bloc — explicit event/state traces, strong for complex flows and team consistency; `ChangeNotifier`/Provider — fine for small apps, becomes a tangle at scale). **The real rule is consistency:** one approach per app, applied everywhere. Mixed approaches are worse than any single choice.
- The `setState`-in-a-1500-line-widget antipattern and how to decompose it.
- Repositories return domain entities, not DTOs or raw JSON. A repository that just forwards to an HTTP client is doing nothing — say so.
- Error handling: typed results (`Result`/`Either`/sealed classes) at layer boundaries rather than exceptions crossing layers. Every async operation has a defined failure path.
- **Offline is a first-class requirement for mobile + desktop, not a stretch goal.** Define the local store (Drift/Isar/sqflite/Hive), the source-of-truth policy, and the conflict resolution rule. "It requires network" is a design decision that must be stated, not defaulted into.
- Navigation: declarative routing (`go_router`) with typed routes, deep links, and — critically — **desktop-appropriate navigation.** A bottom nav bar on a 1440px Windows window is wrong. Branch on form factor.
- **Absorb the PlannedTasks-generation workflow** from the existing skill but note it now belongs to `core-eng/spec-writer` — this skill keeps only the architecture rules.

**Absorb:** `/mnt/skills/user/flutter-fullstack` (architecture portions; planning/PlannedTasks portions → `core-eng/spec-writer`).

**reference/:** `layering.md` — directory template and layer-boundary rules with examples.

**Budget:** 200 lines.

---

## `flutter-ui-craft` — CORE

**description:** `Build Flutter screens and widgets with a deliberate visual system — theming, typography, spacing, adaptive layout across mobile and desktop. Do NOT use for animation, that's animate/flutter-motion-craft.`

**Owns:** widget composition, `ThemeData` and token systems, responsive/adaptive layout, component library, rendering every UI state.
**Does NOT own:** motion → `animate/flutter-motion-craft` + `flutter-animation-system`; app structure → `flutter-architecture`; visual review → `critique/ui-critique`.

**Body must cover:**
- **[R4] Read `/mnt/skills/public/frontend-design/SKILL.md` first.** It supplies aesthetic direction; this skill supplies Flutter execution. Default Material 3 with a seed colour is the Flutter equivalent of the templated look — say so explicitly and design past it.
- **Centralised theme is non-negotiable.** All colour, type, spacing, radius, and elevation come from `ThemeData` extensions (`ThemeExtension<T>`). A hardcoded `Color(0xFF...)` or `EdgeInsets.all(16)` in a widget is a defect. This mirrors `web-ui/design-system`'s token layering — reference it for the concept, implement in Dart.
- Typography: a named `TextTheme` with role names, not sizes at call sites. Custom fonts via `google_fonts` or bundled assets, with explicit fallbacks.
- **Adaptive, not just responsive.** Three tiers: compact (<600dp, phone), medium (600–1024dp, tablet/small window), expanded (>1024dp, desktop). Navigation, density, and interaction change per tier — bottom nav → nav rail → nav drawer. Desktop adds hover states, right-click menus, keyboard shortcuts, tighter density, and resizable panes. A phone layout stretched to 1920px is the most common Flutter desktop failure.
- Widget composition: extract widgets into classes, never into `Widget _buildX()` methods (methods break `const` optimisation and rebuild scoping). Use `const` constructors aggressively.
- **Every screen renders five states:** loading, empty, populated, error, offline. Design the empty state.
- Text scaling to 200% must not clip — check it; most Flutter layouts break here.
- Desktop-specific correctness: no fixed heights that assume a phone, scrollbars visible on desktop (`Scrollbar` with `thumbVisibility`), text selectable where a user would expect it, focus traversal working for keyboard users.
- Performance: `const` widgets, `ListView.builder` for anything unbounded, `RepaintBoundary` around expensive subtrees, no expensive work in `build()`, images sized and cached.

**Absorb:** `/mnt/skills/user/flutter-ui-design`.

**reference/:** `theme-template.md` — `ThemeExtension` token setup and adaptive layout scaffolding.

**Budget:** 200 lines.

---

## `flutter-animation-system` — SUPPORTING

**description:** `Implement Flutter animation mechanics — AnimationController, Tween, curves, Hero, AnimatedBuilder, custom painters, transitions. Pairs with animate/flutter-motion-craft, which decides what should be animated.`

**Owns:** the API and mechanics of Flutter animation.
**Does NOT own:** motion taste, timing decisions, choreography → `animate/flutter-motion-craft`; review → `critique/motion-critique`.

**Body must cover:**
- **State the boundary in the first lines:** load `animate/flutter-motion-craft` for the *decision* (what, how long, what curve, how it should feel). This skill implements it. If craft has already specified the motion, implement it exactly — don't re-decide.
- Decision tree for which API to use: implicit widgets (`AnimatedContainer`, `AnimatedOpacity`, `AnimatedAlign`, `AnimatedSwitcher`, `TweenAnimationBuilder`) → explicit `AnimationController` when you need orchestration, interruption, reversal, or a driven sequence → `SpringSimulation`/physics when the user drags it. Most code reaches for a controller too early.
- `AnimationController` lifecycle: `TickerProviderStateMixin` (or `Single`), `dispose()` always, `vsync`, `forward`/`reverse`/`repeat`, `AnimationStatus` listeners. A leaked controller is a real memory bug.
- `AnimatedBuilder` and `AnimatedWidget`: use the `child` parameter to keep static subtrees out of the rebuild. This is the single most impactful Flutter animation performance rule.
- Staggered sequences via `Interval` on one shared controller, not multiple controllers.
- Transitions: `PageRouteBuilder`, `Hero` (including `flightShuttleBuilder` for non-trivial cases), `AnimatedSwitcher` with proper `Key`s — a missing key is why `AnimatedSwitcher` silently does nothing.
- `CustomPainter` for anything the widget tree can't express; `shouldRepaint` implemented correctly or it repaints every frame.
- `flutter_animate` as a shorthand for simple chained effects — useful, but note it hides the controller and can encourage over-animating.
- Profiling: DevTools timeline in **profile mode**. Debug-mode frame times are meaningless. Watch raster vs UI thread — they have different fixes.
- Honour `MediaQuery.disableAnimations` and Android's system animation scale.

**Budget:** 180 lines.

---

## `flutter-desktop-platform` — SUPPORTING

**description:** `Handle Windows and Linux desktop specifics in Flutter — window management, file system, native integration, platform channels, per-platform behaviour differences.`

**Owns:** desktop platform integration, window behaviour, file system access, native channels, platform divergence.
**Does NOT own:** packaging and distribution → `flutter-release`; adaptive layout → `flutter-ui-craft`.

**Body must cover:**
- **Blunt framing:** Flutter desktop is less mature than mobile. Plugin support is uneven, Linux especially. Verify plugin desktop support on pub.dev *before* designing a feature around it — discovering a plugin is Android-only after building the UI is the most common wasted week here.
- Window management (`window_manager` or equivalent): min/max size, remembering size and position across launches, title bar customisation, multi-window caveats, full-screen. A desktop app that opens at a fixed phone-sized window reads as a port.
- File system: `path_provider` gives the correct per-platform directories — use them, never write next to the executable. Windows `%APPDATA%` vs Linux XDG base directories differ; state both. File pickers, drag-and-drop targets, and "open with" associations.
- Keyboard and pointer: `Shortcuts`/`Actions`/`Focus`, standard accelerators per platform, context menus on right-click, hover states, cursor changes, mouse wheel and trackpad handling.
- Platform channels: when needed, how to implement per-platform (C++ for Windows, GTK/C for Linux), and how to fail gracefully when a channel is unavailable.
- Divergence to plan for: Linux distro variance (GTK versions, Wayland vs X11, missing system libraries), Windows Defender/SmartScreen flagging unsigned binaries, DPI scaling differences, font availability.
- Test matrix statement: a build that runs on the dev machine proves nothing. Test on a clean Windows VM and at least two Linux distros.
- Single codebase across Android + Windows + Linux means branching behaviour, not branching codebases — use `Platform`/`defaultTargetPlatform` guards at the edges, keep the core shared.

**Budget:** 150 lines.

---

## `flutter-release` — SUPPORTING

**description:** `Build, sign, package, and distribute Flutter apps for Android, Windows, and Linux — store submission, installers, CI, versioning. Use for shipping, not for the go/no-go decision — that's secure-ship/prod-readiness-gate.`

**Owns:** build configuration, signing, packaging, store/distribution mechanics, release CI.
**Does NOT own:** release gating → `secure-ship/prod-readiness-gate`; platform runtime behaviour → `flutter-desktop-platform`.

**Body must cover:**
- Build flavours and environment config per flavour (dev/staging/prod) using `--dart-define-from-file`, not committed config files.
- **Android:** app bundle (`.aab`) for Play, keystore stored in CI secrets only and never committed, ProGuard/R8 rules for reflection-using packages, `--obfuscate --split-debug-info` with symbols archived for crash de-obfuscation, target SDK compliance deadlines, Play Console data-safety form.
- **Windows:** MSIX packaging (`msix` package) for Store or sideload, code signing certificate — state plainly that unsigned Windows binaries trigger SmartScreen warnings and materially hurt install rates, so budget for a certificate. Bundle the correct VC++ runtime. Installer alternative: Inno Setup / MSI.
- **Linux:** choose the distribution channel deliberately — `.deb` (Debian/Ubuntu, simplest), Flatpak (broadest reach, sandbox constraints to design around), Snap (Ubuntu-centric), AppImage (portable, no install). Each has different dependency handling; name the tradeoffs. Bundle shared libraries or declare them correctly, or it fails on a clean machine.
- Versioning: one source of truth in `pubspec.yaml` (`version: x.y.z+build`), propagated to all three platforms. Build numbers must increase monotonically for Play.
- CI: build all three targets on every release tag. Windows and Linux builds require their own runners — a mac/linux-only pipeline silently stops testing Windows.
- Crash reporting (Sentry/Crashlytics) initialised before `runApp`, with symbol upload wired into CI, and verified with a deliberate test crash.
- Update strategy: Play handles Android; desktop needs an explicit answer (in-app updater, store, or manual) — decide it rather than defaulting to nothing.

**reference/:** `packaging.md` — per-platform build commands and CI workflow templates.

**Budget:** 170 lines.
