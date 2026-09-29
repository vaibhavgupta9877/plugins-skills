# Plugin: `modernize`

**Enable for:** any project where an existing web page or Flutter screen must be brought to current S-tier quality in one pass.
**Purpose:** a single entry point that sequences the other eight plugins — audit, direction, plan, build, verify, report — so no dimension is silently skipped. It owns *order and coverage*, never a discipline.
**Skills:** 2 Core.

**Boundary reminder for the agent:** `modernize/*` never restates another skill's rules; it names the skill, the phase, and the section to load. Single-discipline requests go straight to the owning skill (`web-ui/landing-page-craft`, `flutter/flutter-ui-craft`, `critique/*`, …). Building a brand-new page from nothing is `landing-page-craft` / `web-design-engineering` / `flutter-ui-craft`, not this plugin.

**Dependencies:** works best with `product-direction`, `critique`, `animate`, `core-eng`, `secure-ship`, and the stack plugin (`web-ui` or `flutter`) enabled. A missing plugin is reported and its ledger rows marked `Unverified`; the run does not fake the lens.

---

## `web-page` — CORE

**description:** `Use when modernizing an existing web page or app screen end-to-end to S-tier — audit, direction, rebuild, verify. Do NOT use for a single discipline (landing-page-craft, design-system, critique/*) or Flutter (modernize/flutter-screen).`

**Owns:** the modernization sequence for one web page, the coverage ledger, the preservation contract.
**Does NOT own:** any discipline — direction → `product-direction/*`; tokens/structure/copy/perf → `web-ui/*`; states/motion → `animate/*`; judgement → `critique/*`; review/tests/commits → `core-eng/*`; ship gate → `secure-ship/*`.

**Body must cover:**
- **[R4]** `frontend-design` directive first.
- Phase 0 recon (stack detection, surface class, direction on file, baseline screenshots + perf, preservation list) before any question.
- Phase 1 baseline audit through the critique lenses + design-system audit mode + perf measurement, scored on `reference/s-tier-scorecard.md`.
- Phase 2 direction gate with explicit routing conditions and a single AskUserQuestion intake batch (ambition tier: Refresh / Redesign / Signature).
- Phase 3 plan (finding → change → owning skill → tier), one approval.
- Phase 4 fixed build order: tokens → structure → copy → interaction states → motion → modern platform pass → perf + SEO.
- Phase 5 verification: re-run lenses, re-score, same-condition after-evidence, code review, tests, ship gate.
- Phase 6 report with a **coverage ledger** in which every dimension ends `Done | N/A | Deferred | Unverified`.

**reference/:** `s-tier-scorecard.md` (5 gates, 15 dimensions, evidence rules); `modern-web-platform.md` (current platform features as progressive enhancement, with fallbacks).

**Budget:** 200 lines.

---

## `flutter-screen` — CORE

**description:** `Use when modernizing an existing Flutter screen or flow end-to-end to S-tier — audit, direction, rebuild, verify across platforms. Do NOT use for a single discipline (flutter-ui-craft, flutter-motion-craft, critique/*) or web (modernize/web-page).`

**Owns:** the modernization sequence for one Flutter screen or short flow on every in-scope platform, the coverage ledger, the preservation contract.
**Does NOT own:** any discipline — direction → `product-direction/*`; theme/layout/states → `flutter/flutter-ui-craft`; structure → `flutter/flutter-architecture`; desktop → `flutter/flutter-desktop-platform`; motion → `animate/flutter-motion-craft` + `flutter/flutter-animation-system`; copy → `web-ui/ux-copy`; judgement → `critique/*`.

**Body must cover:**
- **[R4]** `frontend-design` directive first.
- SDK-gated advice: detect `pubspec.yaml` constraint and `flutter --version`; never suggest APIs beyond it without proposing the bump.
- Baseline per platform × width tier × theme × text scale; profile-mode frame timings only.
- Same phase structure as `web-page`, with the Flutter build order: decompose → theme/tokens → adaptive structure → five states → copy → interaction states → motion (craft decides, system implements) → modern Flutter pass → performance.
- Verification includes text scale 2.0, disabled animations, platform screen readers, desktop keyboard traversal, `flutter analyze`, widget + golden tests.
- Ledger rows are per dimension **and** per platform.

**reference/:** `s-tier-scorecard.md` (5 gates, 14 dimensions, evidence rules); `modern-flutter-platform.md` (current APIs, deprecations to retire, platform expectations).

**Budget:** 200 lines.

## Sources

Orchestration of this marketplace's existing plugins; `frontend-design` (Anthropic). Scorecard bars come from WCAG 2.1 AA, Core Web Vitals thresholds, Flutter profiling guidance, and `product-direction/skills/art-direction/reference/anti-slop.md`.
