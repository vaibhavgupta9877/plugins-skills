# Plugin: `critique`

**Enable for:** every project, always.
**Purpose:** judgement. Every skill here *reviews* something and produces findings. None of them build.
**Skills:** 6

**The one rule that keeps this plugin from colliding with everything else:** critique skills never write production code. They produce findings — severity, location, why it's wrong, what to change. If the user then says "fix it," control passes to the relevant building skill.

**Shared output contract (state it in every body):**
Findings ranked by severity — `blocking` / `should-fix` / `nit`. Each finding: what · where · why it matters · concrete fix. End with **the single most important change**. Never pad with manufactured findings; "this is solid, here are two nits" is a valid and valuable output.

When `docs/product-direction.md` exists, `ui-critique` uses verified Brand foundation and matching surface context when present, while keeping the findings-only contract. If no surface section exists, it reviews available evidence, states any scope limit, and does not infer a direction violation. `product-direction` defines direction; critique judges the rendered result.

---

## `ui-critique` — CORE

**description:** `Review a UI screen, mockup, or component for hierarchy, spacing, typography, consistency, and whether it reads as templated. Do NOT use to review motion — that's motion-critique.`

**Owns:** visual design review, layout, hierarchy, type, colour, spacing rhythm, "does this look AI-generated."
**Does NOT own:** motion → `motion-critique`; accessibility → `a11y-review`; interface wording → `copy-critique`; building the fix → `web-ui/*` or `flutter/flutter-ui-craft`.

**Body must cover:**
- **[R4] First read `/mnt/skills/public/frontend-design/SKILL.md`.** Judge against its standard: is there a real point of view, or is this the templated default?
- Use the three AI-design tells as diagnostic prompts, not bans: warm-cream + high-contrast serif + terracotta accent; near-black + single acid accent; broadsheet hairline-rule grid. A resemblance without product-specific rationale is `should-fix`, not automatically **blocking**. Reserve blocking for verified-direction conflicts or material task, trust, readability, accessibility, or performance harm; fake proof remains blocking because it is false evidence.
- Check order: does the hierarchy match the actual priority of the content → is spacing on a consistent scale → is the type scale deliberate → is colour doing work or decorating → is there a signature element, and is exactly one thing bold while everything else stays quiet.
- Consistency sweep: same component, different spacing/radius/shadow across screens is the most common real defect.
- Works for web screenshots, Figma descriptions, Flutter widget trees, and live code.
- Say what's *working* first, briefly. Reviews that only attack get ignored.

**Absorb:** `/mnt/skills/plugins/design:design-critique`, `/mnt/skills/user/design-system` (review portions only).
**Sources:** `github.com/Owl-Listener/designer-skills/tree/main/visual-critique`, `github.com/jakubkrehel/skills` (`interface-review`).

**Budget:** 180 lines.

---

## `motion-critique` — CORE

**description:** `Review existing animations for timing, easing, purpose, choreography, and performance. Use to judge motion that already exists. Do NOT use to add or improve animation — that's animate/improve-animations.`

**Owns:** judging motion quality across web and Flutter.
**Does NOT own:** creating or fixing motion → `animate/*`.

**Body must cover:**
- **[R4] First read `/mnt/skills/public/frontend-design/SKILL.md`** — specifically its restraint guidance. Extra animation is a primary AI-generated tell.
- Load `animate/animation-vocabulary` for shared terminology so findings are precise rather than "feels off."
- Check order: **purpose** (does this motion carry information, or is it decoration) → **duration** (most AI-authored motion is 2–3× too slow; enter 150–250ms, exit faster) → **easing** (linear and symmetric ease-in-out are the two most common defects; exits should not ease-in) → **choreography** (is there one orchestrated moment, or scattered independent effects) → **interruption** (what happens if the user acts mid-animation — does it queue, jump, or gracefully redirect) → **performance** (is it compositor-only on web; is it rebuilding the widget tree on Flutter).
- `prefers-reduced-motion` unhandled is a **blocking** finding on web. Flutter equivalent: `MediaQuery.disableAnimations`.
- Scroll-linked motion: check it doesn't hijack scroll, doesn't break on trackpad vs wheel vs touch, and degrades if JS fails.
- The hardest and most valuable finding to make: "delete this animation." Make it when it's true.

**Sources:** `github.com/emilkowalski/skills` (`review-animations`).

**Budget:** 170 lines.

---

## `a11y-review` — SUPPORTING

**description:** `Audit a UI against WCAG 2.1 AA — contrast, keyboard navigation, focus order, semantics, touch targets, screen reader behaviour. Do NOT use for visual design quality — that's ui-critique.`

**Owns:** accessibility conformance findings, web and Flutter.
**Does NOT own:** aesthetics → `ui-critique`; motion preferences → `motion-critique` (overlaps on reduced-motion; a11y-review defers).

**Body must cover:**
- Check order: contrast ratios (4.5:1 body, 3:1 large text and UI boundaries) → keyboard reachability and visible focus → focus order matches visual order → semantic elements not `div`s with handlers → labels on every input → touch targets ≥44×44 → images have meaningful alt or are marked decorative → live regions for async state.
- Flutter specifics: `Semantics` widgets, `excludeSemantics`, `MergeSemantics`, TalkBack testing, text scaling to 200% without clipping (this breaks most Flutter layouts — check it).
- Next.js specifics: focus management on route change, skip links, modal focus trapping and restoration, `aria-live` for toasts and tRPC mutation states.
- Distinguish conformance failures (blocking) from usability improvements (should-fix). Don't inflate.

**Absorb:** `/mnt/skills/plugins/design:accessibility-review`.

**Budget:** 120 lines.

---

## `architecture-critique` — CORE

**description:** `Audit a system or module's structure — boundaries, coupling, data flow, state ownership, scalability. Use for whole-design review. Do NOT use for PR diffs — that's core-eng/code-review.`

**Owns:** structural review, coupling/cohesion, layering violations, state ownership, extensibility.
**Does NOT own:** diff review → `core-eng/code-review`; API surface shape → `api-critique`; security → `secure-ship/threat-model`.

**Body must cover:**
- Map the actual dependency direction before judging. Most architecture problems are direction problems: UI reaching into data, or a domain layer importing a framework.
- Check order: are boundaries real or nominal → does state have exactly one owner → is data flow unidirectional or does it loop → what would it cost to replace any one layer → where is business logic actually living (usually smeared across the UI).
- Per-stack failure patterns: Flutter (logic in widgets, `BuildContext` leaking into domain code, state management chosen but not followed consistently, repository layer that's a thin pass-through doing nothing), Next.js T3 (server/client boundary blurred, Prisma calls in components, tRPC routers that mirror DB tables instead of use-cases, "utils" as a dumping ground), Rust (over-generic APIs with lifetime parameters nobody needs, `pub` on everything, trait abstractions with one implementor).
- Judge against the *actual* scale, not a hypothetical one. Over-architecture for a scale you'll never reach is a finding, and a common one.
- Always answer: what is the one structural change with the highest payoff-to-disruption ratio.

**Budget:** 180 lines.

---

## `api-critique` — SUPPORTING

**description:** `Review a public API surface — REST/tRPC endpoints, Rust crate public items, Dart package exports — for ergonomics, consistency, and evolvability. Do NOT use for internal implementation review.`

**Owns:** public surface review, naming consistency, error contracts, versioning risk, breaking-change detection.
**Does NOT own:** designing a new API → `next-saas/api-design`, `rust-crate/api-ergonomics`; auth → `secure-ship/threat-model`.

**Body must cover:**
- The test that matters: can a competent stranger use this correctly on first attempt, using only the type signatures and docs?
- Check order: naming consistency across the whole surface → is the error contract uniform and typed → are optional/required boundaries honest → is anything `pub`/exported that shouldn't be → what becomes a breaking change later.
- **Rust public crates:** every `pub` item is a permanent commitment. Flag leaked internal types, missing `#[non_exhaustive]` on enums and structs likely to grow, generic parameters that force turbofish at call sites, and error types that don't implement `std::error::Error`.
- **tRPC:** procedures should express use-cases, not CRUD over tables. Flag input schemas that accept more than the procedure uses, and mutations that return nothing useful.
- Detect breaking changes explicitly against the previous version when one exists.

**Budget:** 130 lines.

---

## `copy-critique` — THIN

**description:** `Review interface copy — labels, buttons, errors, empty states, onboarding — for clarity and voice consistency. Do NOT use to write new copy, that's web-ui/ux-copy.`

**Owns:** judging existing interface text.
**Does NOT own:** authoring copy → `web-ui/ux-copy`; marketing copy → `web-ui/landing-page-craft`.

**Body must cover:**
- **[R4] Read `/mnt/skills/public/frontend-design/SKILL.md`**, "More on writing in design" section — it is the standard. Judge against it directly.
- Highest-value checks: does a control name what happens ("Save changes" not "Submit") · does the action keep its name through the whole flow · do errors say what broke *and* how to fix it · are empty states an invitation or a dead end · is anything named after implementation rather than user intent.
- Voice consistency across screens matters more than any individual line.

**Budget:** 60 lines.
