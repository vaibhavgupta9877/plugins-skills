---
name: flutter-screen
description: Use when modernizing an existing Flutter screen or flow end-to-end to S-tier — audit, direction, rebuild, verify across platforms. Do NOT use for a single discipline (flutter-ui-craft, flutter-motion-craft, critique/*) or web (modernize/web-page).
---

# modernize/flutter-screen

> **Read first.** Invoke the `frontend-design` skill (plugin `frontend-design@claude-plugins-official`;
> on Linux sandboxes, `/mnt/skills/public/frontend-design/SKILL.md`) and follow it. It is the source of
> truth for aesthetic direction, typography, palette, restraint, and interface copy. This skill only
> sequences the other skills on top of it and never contradicts it.

## Owns
- The **order of work** for taking one existing Flutter screen or short flow (home, onboarding,
  dashboard, detail, settings, paywall) to current S-tier on every target platform:
  recon → audit → direction → plan → build → verify → report.
- The **coverage ledger** — proof that no dimension or platform was silently skipped.
- The preservation contract — what a modernization must not break.

## Does not own
Every discipline is delegated. This skill never re-states their rules; it calls them.
- Brand, design language, surface brief → `product-direction/*`. Reference study → host `tastemaker`.
- Theme, widgets, adaptive layout, states → `flutter/flutter-ui-craft`. Structure and state
  management → `flutter/flutter-architecture`. Windows/Linux specifics → `flutter/flutter-desktop-platform`.
- Motion decision → `animate/flutter-motion-craft`; motion API → `flutter/flutter-animation-system`.
- Interface copy → `web-ui/ux-copy` (its patterns are stack-agnostic). Judgement → `critique/*`.
- Diff review, tests, commits → `core-eng/*`. Ship gate → `secure-ship/prod-readiness-gate`.
- Web pages → `modernize/web-page`. A whole-app re-architecture → `flutter/flutter-architecture`.

**How to call a skill:** use the Skill tool with its qualified name (`flutter:flutter-ui-craft`). Load
each skill only in the phase that needs it — never all at once. If a plugin is not enabled, say so,
mark that ledger row `Unverified`, and suggest `/plugin install <plugin>@gupta-skills`.

---

## Phase 0 — Recon (read-only, no questions yet)

1. **Target.** Name the screen(s), their files, routes, and the widgets they own. One screen or one
   short flow per run; shared theme work is done on the first run.
2. **Project facts.** Read `pubspec.yaml` (SDK constraint, state management, router, `animations`,
   `google_fonts`, `flutter_animate`, l10n), `analysis_options.yaml`, and the platform folders present
   (`android/ ios/ windows/ linux/ macos/ web/`). Run `flutter --version`. **API advice must fit the
   detected SDK** — never suggest an API newer than the constraint without proposing the bump.
3. **Theme on file.** Locate `ThemeData`, `ColorScheme`, `TextTheme`, `ThemeExtension`s, and whether
   `useMaterial3` is on. Run `flutter-ui-craft` §2's hardcoded-value greps scoped to the target.
4. **Direction on file.** Read `docs/product-direction.md` (Brand foundation, `Design language`,
   matching surface section) if it exists.
5. **Baseline evidence.** Before-screenshots (or goldens) for each target platform at compact (~390dp),
   medium (~800dp), and expanded (~1440dp window) widths, light + dark, text scale 1.0 and 2.0. A
   profile-mode run (`flutter run --profile`) of the key interactions on a real mid-range device with
   DevTools frame timings. If nothing can be run, say so — verdicts become code-read only, labelled.
6. **Preservation list.** Routes and deep links, analytics events, test `Key`s and semantics labels
   used by integration tests, l10n/ARB keys, state-management contracts (providers/blocs the screen
   reads), platform channels, verified copy and claims, existing accessibility wins.

## Phase 1 — Baseline audit (findings only, no edits)

| Lens | Skill |
|---|---|
| Visual + templated-default check (Flutter widget-tree mode) | `critique:ui-critique` |
| Accessibility (Semantics, targets, contrast, text scale, keyboard) | `critique:a11y-review` |
| Copy | `critique:copy-critique` |
| Existing motion | `critique:motion-critique` — N/A if there is none |
| Screen structure (1500-line `setState` widget, logic in `build`) | `flutter:flutter-architecture` §4 — only the target screen |
| Performance (UI vs raster thread) | `flutter:flutter-animation-system` §9 profiling guidance |

Score the screen with `reference/s-tier-scorecard.md`. The score and top three findings are the
"before" half of the final report.

## Phase 2 — Direction gate

| Condition | Route |
|---|---|
| No usable brand facts | `product-direction:brand-foundation` |
| No `Design language`, **or** the screen reads as stock Material 3 / templated, **or** user wants "premium / not generic" | `product-direction:art-direction` |
| Redesign changes the journey, hierarchy, or navigation | `product-direction:experience-direction` |
| User supplied reference apps/screenshots | host `tastemaker`, when available |

**Intake — one batch, via AskUserQuestion, only what the evidence could not answer:**
- **Ambition tier:** *Refresh* (theme, states, a11y, adaptive, perf; layout kept) · *Redesign*
  (structure, navigation per form factor, one orchestrated motion moment) · *Signature* (premise-led
  visual language, custom painters/shaders or hero transition as the one signature moment).
  Default: Refresh for high-frequency task screens, Redesign for onboarding/home/paywall.
- Which platforms are in scope (default: every platform folder present), and the minimum device.
- Hard constraints: SDK bump allowed? new packages allowed? brand fixed?

Direction choices stay `Assumption` until the user approves them. No approval, no build.

## Phase 3 — Plan

Every audit finding and every scorecard gap → a change → the owning skill → the tier → the platforms
it affects. Order it by the build sequence below. For a flow of more than ~3 screens, hand the plan to
`core-eng:spec-writer`. Show the plan and the ledger skeleton; get one approval, then build.

## Phase 4 — Build, in this order

1. **Decompose if needed** — `flutter:flutter-architecture` §4 order: widgets as classes, logic out of
   `build`, into the project's *existing* state approach. No state-library switch in a UI run.
2. **Theme and tokens** — `flutter:flutter-ui-craft` §1–3 and its `reference/theme-template.md`:
   approved `Design language` values → `ColorScheme` (designed, not seed-only), `TextTheme` roles,
   `ThemeExtension` tokens, component themes with `WidgetStateProperty`. Replace the Phase 0 literals.
3. **Adaptive structure** — `flutter-ui-craft` §4: compact / medium / expanded change navigation,
   density, and interaction, not just width. Windows/Linux window, menus, shortcuts →
   `flutter:flutter-desktop-platform`.
4. **Five screen states** — `flutter-ui-craft` §6: loading skeleton, empty invitation, populated edge
   cases, error from the typed `Failure`, offline with cached data.
5. **Copy** — `web-ui:ux-copy` patterns; never fabricate proof, stats, or reviews.
6. **Interaction states** — hover, press, focus, disabled on every control, set in the theme; desktop
   hover, context menus, visible scrollbars, selectable text (`flutter-ui-craft` §8).
7. **Motion** — `animate:flutter-motion-craft` writes the spec (per-platform table, one orchestrated
   moment); `flutter:flutter-animation-system` implements it exactly. `disableAnimations` honoured
   globally.
8. **Modern Flutter pass** — `reference/modern-flutter-platform.md`: current APIs and platform
   expectations (predictive back, edge-to-edge, `WidgetStateProperty`, granular `MediaQuery.*Of`),
   gated on the detected SDK.
9. **Performance** — `flutter-ui-craft` §9 and `flutter-animation-system` §9; profile mode only.

Re-check the preservation list after steps 1, 3, and 7.

## Phase 5 — Verify

1. Re-run every Phase 1 lens on the new screen. Blocking findings are fixed before continuing.
2. Re-score with `reference/s-tier-scorecard.md`; S-tier means every gate passes and no dimension is
   below its bar — **on every in-scope platform**, not just the one you ran.
3. After-screenshots/goldens with the same widths, themes, and text scales. Plus: text scale 2.0 with
   no clipping, animations disabled, TalkBack (Android) / Narrator (Windows) / Orca (Linux) where
   available, full keyboard traversal on desktop, window resize with no animated layout.
4. `flutter analyze` clean; `core-eng:test-strategy` for widget tests of the five states and golden
   tests for compact/expanded × light/dark; `core-eng:code-review` on the diff;
   `secure-ship:prod-readiness-gate` (Flutter checklist) when shipping.
5. Commit through `core-eng:git-pr` conventions when the user asks for commits.

## Phase 6 — Report

```markdown
## Modernization: <screen/flow>
**Tier:** Refresh | Redesign | Signature · **SDK:** … · **Platforms:** … · **Evidence:** device-run | code-read only

### Before → after
| Dimension | Before | After |   (from the scorecard, plus frame timings and tell notes)

### What changed          (grouped by build step, with files)
### Platform notes        (what differs on Android / Windows / Linux / others, and why)
### Coverage ledger       (every scorecard dimension × platform: Done | N/A — reason | Deferred — approved | Unverified — why)
### Preserved             (each preservation-list item, confirmed)
### Open decisions        (font licences, SDK bump, real content needed, anything still Assumption)
```

## Rules that keep it honest

- **The ledger is the anti-miss mechanism.** Every dimension, on every in-scope platform, ends in
  exactly one state. A missing entry means the run is not finished.
- Debug-mode performance is not evidence. Emulator-only results are labelled as such.
- Never claim a platform was checked that was not run; label code-read verdicts.
- Modernize ≠ trend-chase. Each change traces to a finding, a scorecard gap, or approved direction.
- Keep what works; stop and ask only at the two gates or on a preservation risk.

## Sources
Orchestrates this marketplace's `product-direction`, `flutter`, `animate`, `web-ui/ux-copy`,
`critique`, `core-eng`, and `secure-ship` skills; `frontend-design` (Anthropic). Scorecard bars derive
from WCAG 2.1 AA, Flutter profiling guidance, and `art-direction/reference/anti-slop.md`.
