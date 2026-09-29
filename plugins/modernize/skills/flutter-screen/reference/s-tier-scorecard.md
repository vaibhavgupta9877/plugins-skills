# S-tier Flutter scorecard

Score every dimension **0–3** before and after, **per in-scope platform** where the result differs.
Each score cites evidence (screenshot/golden, `file:line`, DevTools number). A dimension that does
not apply is `N/A — reason`, never silently omitted.

| Score | Meaning |
|---|---|
| 0 | Missing or broken |
| 1 | Stock default — `ColorScheme.fromSeed` + stock components; looks like every Flutter app |
| 2 | Solid — deliberate, consistent, no real defects |
| 3 | S-tier — distinctive, traced to the premise, platform-native where it should be |

**S-tier verdict:** every **Gate** passes on every in-scope platform · no dimension below 2 ·
Hierarchy and Design language at 3 for onboarding/home/paywall surfaces (at least 2 for
high-frequency task screens, where restraint *is* the S-tier answer).

---

## Gates (pass/fail — a single fail blocks the verdict)

| Gate | Pass condition | Checked by |
|---|---|---|
| G1 Accessibility | `Semantics` on custom controls; targets ≥ 48dp on touch; contrast AA in both schemes; text scale 2.0 with no clipping; full keyboard traversal on desktop; `disableAnimations` honoured | `critique/a11y-review` |
| G2 Performance | Profile mode, real mid-range device: no janky frames in the recorded key interactions (build and raster each within the frame budget); no `Opacity`/`saveLayer` on animating subtrees; unbounded lists are lazy | `flutter/flutter-animation-system` §9 |
| G3 Honest content | No fabricated stats, reviews, users, or ratings; placeholders flagged | this skill |
| G4 Preservation | Every Phase 0 preservation item confirmed intact | this skill |
| G5 No regressions | `flutter analyze` clean; existing tests pass; Phase 1 "what is working" still works | `core-eng/test-strategy` |

---

## Dimensions

| # | Dimension | Score-3 bar | Common score-1 tell |
|---|---|---|---|
| D1 | **Hierarchy** | The primary task or message is found first on every form factor; one primary action per view | Equal-weight cards; FAB + button competing |
| D2 | **Design language** | Premise-led; past the Material default; anti-slop tells 0–2 (`product-direction/skills/art-direction/reference/anti-slop.md`) | Seed purple, stock `Card`s, default `AppBar` |
| D3 | **Typography** | `TextTheme` roles only at call sites; bundled display face with character + workhorse; height/tracking in the role; tabular figures for changing numbers | `TextStyle(fontSize: 19)` inline |
| D4 | **Colour and theming** | Designed `ColorScheme` for light and dark; semantic `ThemeExtension` tokens; rationed accent; dark theme is designed, not inverted | Only `fromSeed`; greys picked per widget |
| D5 | **Layout and rhythm** | One spacing scale; content width capped on wide screens; density differs by tier deliberately | Phone column stretched to 1440dp |
| D6 | **System integrity** | Zero `Color(0x…)` / numeric `EdgeInsets` / inline `TextStyle` literals in touched widgets; component themes set once | Per-widget styling |
| D7 | **States** | Screen: loading skeleton / empty invitation / populated edge cases / typed error with retry / offline with cached data. Controls: hover / press / focus / disabled via `WidgetStateProperty` | Centred spinner; `e.toString()` error |
| D8 | **Copy** | Controls name the action; errors say what + fix; one term per concept; l10n-ready (no concatenation) | "Submit", "Oops!" |
| D9 | **Motion** | Spec per platform (Android vs desktop timings); one orchestrated moment; implicit animations first; exits faster; reduced-motion path reaches the end state | `Curves.easeInOut` everywhere; Hero on every list |
| D10 | **Adaptive** | Compact → medium → expanded change navigation (bottom bar → rail → drawer/sidebar), density, and input; `LayoutBuilder` on local constraints | Bottom nav on a desktop window |
| D11 | **Platform fit** | Android: predictive back, edge-to-edge insets, system bars styled, haptics where meaningful. Desktop: visible scrollbars, selectable text, context menus, shortcuts, hover, no animated resize | Content under the gesture bar; unselectable desktop text |
| D12 | **Resilience** | 60-char names, 10,000 items, missing images, slow network, offline all handled; no overflow stripes at any width or text scale | Yellow-black overflow at 2.0× |
| D13 | **Modern Flutter** | Current APIs for the detected SDK, no deprecated calls in touched code (`modern-flutter-platform.md`) | `WillPopScope`, `withOpacity`, `MaterialStateProperty` |
| D14 | **Craft detail** | Optical alignment on icons; one icon set and weight; images sized with `cacheWidth`; launcher icon and splash on-brand; focus rings visible and styled | Mixed icon sets; blurry avatars |

---

## Evidence rules

- A score without evidence is not a score. Code-read-only scores are labelled `(code-read)`;
  emulator-only scores are labelled `(emulator)`.
- Before and after use the same widths, themes, text scales, device, and build mode.
- Performance is profile or release mode only. Debug-mode numbers are discarded, not reported.
- A platform that could not be run is `Unverified` for that platform — never inherited from another.
