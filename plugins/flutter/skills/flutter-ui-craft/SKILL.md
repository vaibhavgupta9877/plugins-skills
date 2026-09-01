---
name: flutter-ui-craft
description: Build Flutter screens and widgets with a deliberate visual system — theming, typography, spacing, adaptive layout across mobile and desktop. Do NOT use for animation, that's animate/flutter-motion-craft.
---

# flutter-ui-craft

> **Read first.** Invoke the `frontend-design` skill (plugin `frontend-design@claude-plugins-official`;
> on Linux sandboxes, `/mnt/skills/public/frontend-design/SKILL.md`) and follow it. It supplies the
> aesthetic direction; this skill supplies the Flutter execution and never contradicts it.

## Owns
- Widget composition, `ThemeData` and token systems, adaptive layout, component library.
- Rendering every UI state.

## Does not own
- Motion → `animate/flutter-motion-craft` (taste) + `flutter-animation-system` (API).
- App structure and state management → `flutter-architecture`.
- Desktop platform integration → `flutter-desktop-platform`.
- Visual review → `critique/ui-critique`.

---

## 1. Design past the Material default

**Default Material 3 with a seed colour is the Flutter equivalent of the templated look.** It is a
sensible starting point and a recognisable one: `ColorScheme.fromSeed` plus stock components produces
an app that looks like every other Flutter app, which is exactly what `frontend-design` warns
against.

Take the direction from `frontend-design` — palette, typefaces, the one signature element — and then
implement it in `ThemeData`. Restyle the components; do not ship the defaults.

---

## 2. Centralised theme is non-negotiable

**All colour, type, spacing, radius, and elevation come from the theme.** A hardcoded
`Color(0xFF3B82F6)` or `EdgeInsets.all(16)` in a widget file is a defect.

This mirrors `web-ui/design-system`'s three-layer token model — primitive → semantic → component —
implemented in Dart via `ThemeExtension`:

```dart
@immutable
class AppTokens extends ThemeExtension<AppTokens> {
  final Color surface, surfaceRaised, textMuted, danger;
  final double spaceSm, spaceMd, spaceLg, radiusCard;
  // copyWith + lerp required by ThemeExtension
}

// access
final t = Theme.of(context).extension<AppTokens>()!;
```

Add a `BuildContext` extension (`context.tokens`, `context.colors`, `context.text`) so call sites stay
short — verbosity is why people hardcode.

**Semantic names, not appearance names.** `textMuted`, not `grey600`; the value changes per theme
while the name stays true. See `reference/theme-template.md`.

Audit an existing codebase with:
```bash
rg "Color\(0x" lib/ --glob '!lib/core/theme/**'
rg "EdgeInsets\.(all|symmetric|only)\(\s*[0-9]" lib/ --glob '!lib/core/theme/**'
```

---

## 3. Typography

A named `TextTheme` with **role names, not sizes, at call sites**: `displayLarge`, `headlineMedium`,
`titleMedium`, `bodyLarge`, `bodySmall`, `labelSmall`. `Text('x', style: TextStyle(fontSize: 19))` in
a widget means the scale is missing a role.

Custom fonts via `google_fonts` (convenient; check it is not fetching at runtime in production) or
bundled assets (predictable, offline-safe — preferred for a shipped app). Declare explicit fallbacks
so a missing glyph does not render as boxes on Linux.

Line height (`height`) and letter spacing belong in the role, not at the call site.

---

## 4. Adaptive, not just responsive

Three tiers, and **each changes navigation, density, and interaction — not just widths**:

| Tier | Width | Navigation | Density | Interaction |
|---|---|---|---|---|
| Compact | < 600dp | Bottom nav | Roomy, thumb-reachable | Touch only |
| Medium | 600–1024dp | Navigation rail | Moderate | Touch + pointer |
| Expanded | > 1024dp | Drawer / sidebar, multi-pane | Tighter | Pointer, keyboard, hover |

**A phone layout stretched to 1920px is the most common Flutter desktop failure.** Symptoms: a
full-width single column of cards, a bottom nav bar on a desktop window, 48dp touch targets in a
mouse-driven tool, and text lines 200 characters long.

- Cap content width on wide screens (`ConstrainedBox(maxWidth: 720)` for reading, wider for tables).
- Use `LayoutBuilder` on the **local constraints**, not `MediaQuery` on the window — a widget inside a
  sidebar should respond to the sidebar. This is Flutter's equivalent of container queries.
- Expanded tier earns a master-detail layout instead of push navigation.
- Desktop adds: hover states, right-click context menus, keyboard shortcuts, resizable panes,
  visible scrollbars.

---

## 5. Widget composition

- **Extract widgets into classes, never into `Widget _buildX()` methods.** Methods break `const`
  optimisation and keep the rebuild scope as large as the parent. A `const` child widget is skipped
  entirely on rebuild; a method's return value is not.
- **`const` constructors aggressively** — enable `prefer_const_constructors` in `analysis_options`.
- Small widgets with clear names beat one deeply-nested `build`. If you are counting brackets, split.
- Pass callbacks, not controllers, into leaf widgets.
- `ColoredBox` over `Container(color:)`; `SizedBox` over `Container` for spacing — cheaper, clearer.

---

## 6. Five states per screen

**Loading, empty, populated, error, offline.** A screen with only the populated state is incomplete.

- **Loading**: a skeleton shaped like the real layout, not a centred `CircularProgressIndicator`.
- **Empty**: designed as an invitation, with the action that creates the first item. First-run empty
  and filtered-to-nothing empty are different screens.
- **Error**: recoverable in place, with retry next to what failed, and a message from the typed
  `Failure` — not an exception's `toString()`.
- **Offline**: distinct from error. Show cached data with a marker rather than an error screen — see
  `flutter-architecture` on offline policy.
- **Populated**: with one item, 10,000 items, a 60-character name, and a missing image.

---

## 7. Text scaling and accessibility

**Text scaling to 200% must not clip. This breaks most Flutter layouts — check it explicitly**, with
`MediaQuery(data: ...copyWith(textScaler: TextScaler.linear(2.0)))` in a test or the device setting.

The usual cause is a fixed `height:` on a container holding text. Use constraints and let it grow;
`Flexible`/`Expanded` inside rows; `FittedBox` only where truncation is genuinely acceptable.

Also: `Semantics` labels on icon-only buttons, 48dp minimum touch targets, and contrast checked in
both themes. Full audit belongs to `critique/a11y-review`.

---

## 8. Desktop correctness

- No fixed heights that assume a phone viewport.
- **Scrollbars visible on desktop** — `Scrollbar(thumbVisibility: true)`. Desktop users expect to see
  the scroll position; mobile hides it.
- **Text selectable where a user would expect it** — `SelectableText` or `SelectionArea` for content.
  Non-selectable text in a desktop app reads as broken.
- **Focus traversal works with a keyboard**: logical `FocusTraversalGroup` order, visible focus, and
  Enter/Escape doing the obvious thing in dialogs.
- Hover states on everything interactive (`MouseRegion`, or `WidgetStateProperty` in the theme).
- Right-click context menus where the platform convention expects them.

---

## 9. Performance

- `const` widgets everywhere they are possible.
- **`ListView.builder` / `SliverList.builder` for anything unbounded.** A `Column` with mapped
  children builds every row, off-screen included.
- `RepaintBoundary` around expensive or animating subtrees.
- No expensive work in `build()` — no sorting, no formatting a date list, no `Future` creation.
  `build` can run 60 times a second.
- Images sized (`cacheWidth`/`cacheHeight`) and cached; a full-resolution photo in a 48dp avatar is
  a common memory problem.
- Profile in **profile mode on a real device**. Debug-mode performance is meaningless.

## Sources
`frontend-design` (Anthropic). Absorbed from `flutter-ui-design`: centralised-theme discipline, the
no-hardcoded-values rule, five states, `const`/lazy-list performance rules, cross-platform
intentionality. Its "every state transition must be animated" stance is deliberately not carried
over — it contradicts `frontend-design`'s restraint guidance; motion decisions belong to
`animate/flutter-motion-craft`. Its mandatory package list is also dropped in favour of detecting the
project's actual dependencies.
