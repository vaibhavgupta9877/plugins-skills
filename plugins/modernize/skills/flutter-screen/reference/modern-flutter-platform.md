# Modern Flutter pass

Current Flutter APIs and platform expectations that make a screen feel native in 2026. Rules:

- **Gate every row on the detected SDK** (`pubspec.yaml` environment + `flutter --version`). If an API
  needs a newer SDK, propose the bump as an Open decision — never silently raise the constraint.
- Confirm the exact API against the installed SDK's docs at run time; this file names what to look
  for, not a pinned version table.
- Adopt a row only where the screen has the problem it solves.

## Theming

| Use | Instead of | Why |
|---|---|---|
| Hand-designed `ColorScheme` (or `fromSeed` with a `dynamicSchemeVariant` that keeps the brand hue, then overridden roles) | Bare `ColorScheme.fromSeed(seedColor:)` | Seed-only output is the templated Flutter look |
| `ThemeExtension<T>` with `copyWith`/`lerp` for semantic tokens | Constants files and literals | Theme animation and dark mode stay correct |
| Component themes (`FilledButtonThemeData`, `NavigationBarThemeData`, `InputDecorationTheme`, …) | Styling at each call site | One source of truth per component |
| `WidgetStateProperty` / `WidgetState` | `MaterialStateProperty` (deprecated alias) | Current API; hover/press/focus in the theme |
| `Color.withValues(alpha: …)` | `Color.withOpacity` (deprecated) | Current API, wide-gamut-safe |
| Dynamic colour (Android 12+) **only if the brand allows it** | Assuming it is always desirable | It overrides brand colour |

## Layout and structure

| Use | Instead of |
|---|---|
| `MediaQuery.sizeOf` / `paddingOf` / `viewPaddingOf` / `textScalerOf` / `disableAnimationsOf` | `MediaQuery.of(context).x` (rebuilds on every MediaQuery change) |
| `LayoutBuilder` on local constraints for adaptive components | Window-size checks deep in the tree |
| `NavigationBar` / `NavigationRail` / `NavigationDrawer` switched by width tier | One navigation chrome everywhere |
| Slivers: `CustomScrollView`, `SliverList.builder`, sliver groups and pinned/floating header slivers where the SDK has them | Nested scroll views, `Column` inside `SingleChildScrollView` for long lists |
| `SearchAnchor`, `MenuAnchor`, `SegmentedButton`, `Badge`, `CarouselView` (when available) | Hand-rolled equivalents |
| `SafeArea` / inset padding for **edge-to-edge** (enforced on recent Android target SDKs) | Content under the status/gesture bar |

## Navigation and platform behaviour

| Use | Why |
|---|---|
| `PopScope` (with `onPopInvokedWithResult` where available) | Replaces deprecated `WillPopScope`; required for Android predictive back |
| Predictive back page transitions on Android (and the manifest opt-in) | Expected system gesture feel on Android 14+ |
| `Shortcuts` / `Actions` / `CallbackShortcuts`, `MenuBar` / `PlatformMenuBar` | Desktop keyboard and menu conventions |
| `SelectionArea`, `contextMenuBuilder`, `Scrollbar(thumbVisibility: true)` on desktop | Desktop users expect selection, context menus, and visible scroll |
| `HapticFeedback` on meaningful confirmations (Android) | Physical feedback; never on every tap |

## Text and accessibility

| Use | Instead of |
|---|---|
| `TextScaler` / `MediaQuery.textScalerOf` | `textScaleFactor` (deprecated) |
| Bundled font assets with declared fallbacks; `FontFeature.tabularFigures()` for changing numbers; `FontVariation` for variable fonts | Runtime font fetching in production |
| `Semantics`, `MergeSemantics`, `ExcludeSemantics`; `MediaQuery.highContrastOf` / `boldTextOf` respected | Unlabelled `GestureDetector`s |

## Motion and rendering

| Use | Note |
|---|---|
| Implicit animations and `*Transition` widgets first | Per `animate/flutter-motion-craft` |
| `animations` package transitions (`SharedAxisTransition`, `FadeThroughTransition`, `OpenContainer`) | Material motion patterns on Android; shorten for desktop |
| `FragmentProgram` shaders / `CustomPainter` | Only for the one Signature-tier moment, with a static fallback |
| Test on **Impeller** where it is the platform default; check the renderer per platform | Shader jank and raster costs differ by renderer |

## Retire on sight

`useMaterial3: false` · `WillPopScope` · `MaterialStateProperty` · `withOpacity` · `textScaleFactor` ·
`MediaQuery.of(context).size` in widgets that only need size · `Widget _buildX()` helper methods ·
`Opacity` wrapping animating subtrees · `Column(children: list.map(...))` for unbounded data ·
fixed `height` on text containers · `print` for error handling · centred `CircularProgressIndicator`
as the only loading state.
