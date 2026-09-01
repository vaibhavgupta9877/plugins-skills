---
name: flutter-architecture
description: Structure a Flutter application — layering, state management, dependency injection, routing, data and repository layers, offline behaviour. Do NOT use for widget-level UI styling, that's flutter-ui-craft.
---

# flutter-architecture

## Owns
- Project structure, layering, state management application, DI, routing.
- Repositories, local persistence, offline and sync strategy.

## Does not own
- Visual and widget design → `flutter-ui-craft`.
- Animation API → `flutter-animation-system`; motion taste → `animate/flutter-motion-craft`.
- Desktop platform specifics → `flutter-desktop-platform`.
- Task plans and phased specs → `core-eng/spec-writer`.

---

## 1. Detect before you propose

**Read `pubspec.yaml` and the `lib/` tree before proposing anything.** Then follow what is already
there.

- **Never silently pick Riverpod over Bloc, or `go_router` over Navigator 2.0 directly.** An existing
  choice, applied consistently, beats a better choice applied to 30% of the app.
- If the project is new and the choice is genuinely open, **state the tradeoff and ask** — or hand
  off to `core-eng/grill-me`.
- If the project is inconsistent, that inconsistency is the finding. Propose converging on the
  dominant approach, incrementally.

---

## 2. Layering

```
presentation  →  domain  ←  data
(widgets,        (entities,    (repository impls,
 controllers)     use-cases,    data sources, DTOs)
                  repo interfaces)
```

**Dependencies point inward only.** `domain` imports nothing from `presentation` or `data` — it
defines the repository *interface*, and `data` implements it.

**`BuildContext` never crosses out of `presentation`.** A use-case or repository taking a
`BuildContext` has coupled your business rules to the widget tree, and it cannot be tested or reused.
Flag every violation. The same applies to `MediaQuery`, `Theme`, and navigation calls.

DTOs stay in `data`. If a JSON field name appears in a widget, the layering is nominal, not real.

See `reference/layering.md` for the directory template and worked boundary examples.

---

## 3. Feature-first directories

```
lib/
  features/
    invoices/
      presentation/   # screens, widgets, controllers/notifiers
      domain/         # entities, use-cases, repository interface
      data/           # repository impl, remote + local data sources, DTOs
  core/               # genuinely shared: theme, router, errors, network client, DI
  main.dart
```

**Layer-first layouts (`lib/screens`, `lib/models`, `lib/services`) stop scaling around feature
five** — one change touches four directories, and nothing tells you what the app does. Feature-first
means a feature can be read, reviewed, and deleted as a unit.

`core/` is for what genuinely crosses features. It is not a second dumping ground; if only one
feature uses it, it belongs to that feature.

---

## 4. State management, without dogma

What each is actually good at:

- **Riverpod** — compile-safe DI plus reactive state, testable without a widget tree, good
  granularity. The best default for new work.
- **Bloc** — explicit event→state traces, strong for complex flows and for teams that benefit from
  ceremony and consistency. Verbose for simple screens.
- **`ChangeNotifier` / Provider** — fine for small apps; becomes a tangle at scale because everything
  is mutable and nothing tells you what changed.

**The real rule is consistency: one approach per app, applied everywhere.** Mixed approaches are
worse than any single choice — nobody can predict where state lives, and every screen must be read
from scratch.

Regardless of choice:
- Controllers/notifiers hold **UI state**; business rules live in use-cases or services that can be
  tested without Flutter.
- Model async state as a sealed union (`loading | data | error`), not three loose booleans. Three
  booleans permit eight states, of which five are impossible.
- Dispose everything: controllers, subscriptions, focus nodes, timers.

**The `setState`-in-a-1500-line-widget antipattern.** It is a symptom, not the disease: one widget
owns layout, business rules, network calls, and navigation. Decompose in this order:
1. Extract child widgets **as classes**, not `Widget _buildX()` methods — methods break `const`
   optimisation and keep the rebuild scope enormous.
2. Move network and business logic out into a controller/notifier.
3. Move the rules that don't need Flutter into use-cases, and test them.
4. Only then worry about which state library.

---

## 5. Repositories

**Repositories return domain entities, not DTOs or raw JSON.** The mapping happens in `data`; that
mapping is the repository's main job.

**A repository that just forwards to an HTTP client is doing nothing — say so.** A real repository
owns at least one of: mapping, caching, the offline/online decision, retry policy, or combining
sources. If it owns none of them, delete the layer and call the client.

Interface in `domain`, implementation in `data`. That is what makes the app testable with a fake and
what makes swapping the backend a contained change.

---

## 6. Error handling

**Typed results at layer boundaries, not exceptions crossing layers.** A `Result`/`Either`, or a
sealed failure hierarchy — so the compiler forces the caller to handle failure.

```dart
sealed class Failure { const Failure(); }
class NetworkFailure  extends Failure { const NetworkFailure(); }
class NotFoundFailure extends Failure { const NotFoundFailure(); }
class ServerFailure   extends Failure { final int code; const ServerFailure(this.code); }
```

Catch platform and library exceptions at the edge of `data` and convert. An exception from `dio`
reaching a widget means the boundary does not exist.

**Every async operation has a defined failure path**, and the UI renders it. Not a print, not a
swallowed catch.

---

## 7. Offline is a first-class requirement

For mobile and desktop, **"it requires network" is a design decision that must be stated, not
defaulted into.**

Define three things explicitly:
1. **The local store** — Drift (SQL, relational queries, migrations), Isar/Hive (fast key-value and
   objects), or `sqflite`. Choose on query complexity and migration needs, not on benchmarks.
2. **The source-of-truth policy** — local-first (write local, sync up) or remote-first (cache reads).
   Local-first is what makes an app usable on a train; it is also considerably more work. Pick with
   the user.
3. **The conflict resolution rule** — last-write-wins, server-wins, or per-field merge. Say it out
   loud, because the default is silent data loss.

Also design: what happens to a queued write that fails permanently, how the user sees sync state, and
how a schema migration handles rows written by the previous version.

---

## 8. Navigation

Declarative routing (`go_router`) with typed routes, deep links, and guards for auth. Route state is
owned by the router, not scattered across widgets.

**Desktop-appropriate navigation is not optional.** A bottom navigation bar on a 1440px Windows
window is wrong — it is the clearest signal that a phone app was resized. Branch on form factor:
bottom nav (compact) → navigation rail (medium) → persistent drawer or sidebar (expanded). The
routing stays the same; the chrome changes.

Deep links and "open with" behaviour differ per platform — see `flutter-desktop-platform`.

---

## 9. Restructuring existing code

Incremental, never a rewrite. Establish the convention, apply it to new features, and move existing
code when you touch it for another reason. Sequence by pain: the feature edited weekly moves first.

Planning documents and phased task lists belong to `core-eng/spec-writer`, not here.

## Sources
Absorbed from `flutter-fullstack` (architecture rules; its PlannedTasks workflow now belongs to
`core-eng/spec-writer`). Its hard defaults — Riverpod, GoRouter, Isar, a fixed package list — are
deliberately relaxed to "detect, then be consistent" per this plugin's convention-detection rule.
