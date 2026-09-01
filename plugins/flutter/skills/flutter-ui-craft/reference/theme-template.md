# Theme template — tokens and adaptive scaffolding

Copy, rename, and replace the values with the ones `frontend-design` chose. The structure matters
more than the numbers.

---

## 1. Token extension

```dart
// lib/core/theme/app_tokens.dart
import 'package:flutter/material.dart';

@immutable
class AppTokens extends ThemeExtension<AppTokens> {
  const AppTokens({
    required this.surface,
    required this.surfaceRaised,
    required this.surfaceSunken,
    required this.border,
    required this.textPrimary,
    required this.textMuted,
    required this.accent,
    required this.danger,
    required this.success,
    required this.space,
    required this.radiusSm,
    required this.radiusMd,
    required this.radiusLg,
  });

  // Semantic colour roles — named for meaning, not appearance.
  final Color surface, surfaceRaised, surfaceSunken, border;
  final Color textPrimary, textMuted, accent, danger, success;

  // One spacing scale, indexed. space[2] is not "8px", it is "the second step".
  final List<double> space;
  final double radiusSm, radiusMd, radiusLg;

  @override
  AppTokens copyWith({
    Color? surface, Color? surfaceRaised, Color? surfaceSunken, Color? border,
    Color? textPrimary, Color? textMuted, Color? accent, Color? danger, Color? success,
    List<double>? space, double? radiusSm, double? radiusMd, double? radiusLg,
  }) =>
      AppTokens(
        surface: surface ?? this.surface,
        surfaceRaised: surfaceRaised ?? this.surfaceRaised,
        surfaceSunken: surfaceSunken ?? this.surfaceSunken,
        border: border ?? this.border,
        textPrimary: textPrimary ?? this.textPrimary,
        textMuted: textMuted ?? this.textMuted,
        accent: accent ?? this.accent,
        danger: danger ?? this.danger,
        success: success ?? this.success,
        space: space ?? this.space,
        radiusSm: radiusSm ?? this.radiusSm,
        radiusMd: radiusMd ?? this.radiusMd,
        radiusLg: radiusLg ?? this.radiusLg,
      );

  @override
  AppTokens lerp(covariant AppTokens? other, double t) {
    if (other == null) return this;
    return AppTokens(
      surface: Color.lerp(surface, other.surface, t)!,
      surfaceRaised: Color.lerp(surfaceRaised, other.surfaceRaised, t)!,
      surfaceSunken: Color.lerp(surfaceSunken, other.surfaceSunken, t)!,
      border: Color.lerp(border, other.border, t)!,
      textPrimary: Color.lerp(textPrimary, other.textPrimary, t)!,
      textMuted: Color.lerp(textMuted, other.textMuted, t)!,
      accent: Color.lerp(accent, other.accent, t)!,
      danger: Color.lerp(danger, other.danger, t)!,
      success: Color.lerp(success, other.success, t)!,
      space: space,          // discrete scales don't interpolate
      radiusSm: lerpDouble(radiusSm, other.radiusSm, t)!,
      radiusMd: lerpDouble(radiusMd, other.radiusMd, t)!,
      radiusLg: lerpDouble(radiusLg, other.radiusLg, t)!,
    );
  }
}
```

`lerp` is what makes a theme *animate* rather than snap when the user switches light/dark. Implement
it properly; a `lerp` that returns `this` makes every theme change a hard cut.

---

## 2. Light and dark instances

```dart
// lib/core/theme/tokens_light.dart
const _space = [0.0, 4.0, 8.0, 12.0, 16.0, 24.0, 32.0, 48.0, 64.0];

const appTokensLight = AppTokens(
  surface:       Color(0xFFFFFFFF),
  surfaceRaised: Color(0xFFFFFFFF),
  surfaceSunken: Color(0xFFF6F6F5),
  border:        Color(0xFFE4E3E0),
  textPrimary:   Color(0xFF14140F),
  textMuted:     Color(0xFF6B6A64),
  accent:        Color(0xFF1F6FEB),
  danger:        Color(0xFFB3261E),
  success:       Color(0xFF0F7B45),
  space: _space, radiusSm: 6, radiusMd: 10, radiusLg: 16,
);

const appTokensDark = AppTokens(
  surface:       Color(0xFF14140F),
  surfaceRaised: Color(0xFF1E1E19),   // raised is *lighter* in dark, not shadowed
  surfaceSunken: Color(0xFF0D0D0A),
  border:        Color(0xFF32322C),
  textPrimary:   Color(0xFFF5F4F0),
  textMuted:     Color(0xFF9C9A92),
  accent:        Color(0xFF6DA9FF),   // saturated accents vibrate on dark; lighten and desaturate
  danger:        Color(0xFFF2B8B5),
  success:       Color(0xFF6FD79E),
  space: _space, radiusSm: 6, radiusMd: 10, radiusLg: 16,
);
```

Dark mode is not an inversion. Elevation is expressed by lightening the surface, not by shadow, and
brand colours usually need a lighter, less saturated variant to stay readable.

---

## 3. Context extension — so call sites stay short

```dart
// lib/core/theme/context_ext.dart
extension ThemeX on BuildContext {
  AppTokens   get tokens => Theme.of(this).extension<AppTokens>()!;
  ColorScheme get colors => Theme.of(this).colorScheme;
  TextTheme   get text   => Theme.of(this).textTheme;
}

// call site
Container(
  padding: EdgeInsets.all(context.tokens.space[4]),
  decoration: BoxDecoration(
    color: context.tokens.surfaceRaised,
    borderRadius: BorderRadius.circular(context.tokens.radiusMd),
    border: Border.all(color: context.tokens.border),
  ),
  child: Text('Invoice', style: context.text.titleMedium),
);
```

Verbosity is the reason people hardcode values. Make the correct path the shortest one.

---

## 4. Wiring into `ThemeData`

```dart
ThemeData buildTheme(AppTokens t, Brightness brightness) {
  final base = ThemeData(brightness: brightness, useMaterial3: true);
  return base.copyWith(
    extensions: [t],
    scaffoldBackgroundColor: t.surface,
    colorScheme: base.colorScheme.copyWith(
      surface: t.surface, primary: t.accent, error: t.danger,
    ),
    textTheme: _textTheme(base.textTheme, t),
    // Restyle components — shipping the defaults is what makes apps look identical.
    filledButtonTheme: FilledButtonThemeData(
      style: ButtonStyle(
        shape: WidgetStatePropertyAll(
          RoundedRectangleBorder(borderRadius: BorderRadius.circular(t.radiusSm)),
        ),
        padding: WidgetStatePropertyAll(
          EdgeInsets.symmetric(horizontal: t.space[5], vertical: t.space[3]),
        ),
      ),
    ),
    cardTheme: CardThemeData(
      elevation: 0,
      color: t.surfaceRaised,
      shape: RoundedRectangleBorder(
        borderRadius: BorderRadius.circular(t.radiusLg),
        side: BorderSide(color: t.border),
      ),
    ),
    inputDecorationTheme: InputDecorationTheme(
      filled: true,
      fillColor: t.surfaceSunken,
      border: OutlineInputBorder(
        borderRadius: BorderRadius.circular(t.radiusSm),
        borderSide: BorderSide(color: t.border),
      ),
    ),
  );
}

// app.dart
MaterialApp.router(
  theme:     buildTheme(appTokensLight, Brightness.light),
  darkTheme: buildTheme(appTokensDark,  Brightness.dark),
  themeMode: ThemeMode.system,
);
```

---

## 5. Typography by role

```dart
TextTheme _textTheme(TextTheme base, AppTokens t) => base.copyWith(
  displayLarge:   base.displayLarge!.copyWith(fontSize: 44, height: 1.08, letterSpacing: -0.8, fontWeight: FontWeight.w600),
  headlineMedium: base.headlineMedium!.copyWith(fontSize: 26, height: 1.2,  letterSpacing: -0.3, fontWeight: FontWeight.w600),
  titleMedium:    base.titleMedium!.copyWith(fontSize: 16, height: 1.35, fontWeight: FontWeight.w600),
  bodyLarge:      base.bodyLarge!.copyWith(fontSize: 15, height: 1.55, color: t.textPrimary),
  bodySmall:      base.bodySmall!.copyWith(fontSize: 13, height: 1.5,  color: t.textMuted),
  labelSmall:     base.labelSmall!.copyWith(fontSize: 11, height: 1.3,  letterSpacing: 0.4),
).apply(fontFamily: 'YourFace');
```

Tight leading and negative tracking on large sizes, generous leading on body — the same rule as on
the web. Set it here once so no call site ever passes a raw `fontSize`.

---

## 6. Adaptive scaffolding

```dart
// lib/core/layout/breakpoints.dart
enum FormFactor { compact, medium, expanded }

FormFactor formFactorOf(double width) => width < 600
    ? FormFactor.compact
    : width < 1024
        ? FormFactor.medium
        : FormFactor.expanded;
```

```dart
// lib/core/layout/adaptive_scaffold.dart
class AdaptiveScaffold extends StatelessWidget {
  const AdaptiveScaffold({
    super.key, required this.destinations, required this.selectedIndex,
    required this.onSelect, required this.body, this.detail,
  });

  final List<NavigationDestination> destinations;
  final int selectedIndex;
  final ValueChanged<int> onSelect;
  final Widget body;
  final Widget? detail;               // expanded tier only

  @override
  Widget build(BuildContext context) {
    // LayoutBuilder, not MediaQuery: this must respond to its own constraints,
    // so it still behaves correctly nested inside a pane.
    return LayoutBuilder(builder: (context, constraints) {
      switch (formFactorOf(constraints.maxWidth)) {
        case FormFactor.compact:
          return Scaffold(
            body: body,
            bottomNavigationBar: NavigationBar(
              destinations: destinations,
              selectedIndex: selectedIndex,
              onDestinationSelected: onSelect,
            ),
          );
        case FormFactor.medium:
          return Scaffold(
            body: Row(children: [
              NavigationRail(
                destinations: [
                  for (final d in destinations)
                    NavigationRailDestination(icon: d.icon, label: Text(d.label)),
                ],
                selectedIndex: selectedIndex,
                onDestinationSelected: onSelect,
                labelType: NavigationRailLabelType.all,
              ),
              const VerticalDivider(width: 1),
              Expanded(child: body),
            ]),
          );
        case FormFactor.expanded:
          return Scaffold(
            body: Row(children: [
              SizedBox(width: 240, child: _SidebarNav(destinations, selectedIndex, onSelect)),
              const VerticalDivider(width: 1),
              Expanded(flex: 2, child: body),
              if (detail != null) ...[
                const VerticalDivider(width: 1),
                Expanded(flex: 3, child: detail!),   // master–detail, not push navigation
              ],
            ]),
          );
      }
    });
  }
}
```

The `switch` over an enum is exhaustive — adding a tier later makes the compiler point at every
place that has to handle it.

---

## 7. Reading-width constraint

```dart
class ReadableWidth extends StatelessWidget {
  const ReadableWidth({super.key, required this.child, this.max = 720});
  final Widget child;
  final double max;

  @override
  Widget build(BuildContext context) => Center(
        child: ConstrainedBox(constraints: BoxConstraints(maxWidth: max), child: child),
      );
}
```

Wrap prose and forms. Without it, a maximised desktop window gives you 200-character lines, which is
unreadable and is the most visible symptom of a phone layout on a desktop.

---

## 8. Verification

```bash
# hardcoded values outside the theme directory
rg "Color\(0x" lib/ --glob '!lib/core/theme/**'
rg "EdgeInsets\.(all|symmetric|only)\(\s*[0-9]" lib/ --glob '!lib/core/theme/**'
rg "fontSize:" lib/ --glob '!lib/core/theme/**'
rg "BorderRadius\.circular\(\s*[0-9]" lib/ --glob '!lib/core/theme/**'
```

All four should return nothing outside `lib/core/theme/`. Then check by hand: light and dark, 200%
text scale, a 400dp window, and a 1920px window.
