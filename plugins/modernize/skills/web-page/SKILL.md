---
name: web-page
description: Use when modernizing an existing web page or app screen end-to-end to S-tier — audit, direction, rebuild, verify. Do NOT use for a single discipline (landing-page-craft, design-system, critique/*) or Flutter (modernize/flutter-screen).
---

# modernize/web-page

> **Read first.** Invoke the `frontend-design` skill (plugin `frontend-design@claude-plugins-official`;
> on Linux sandboxes, `/mnt/skills/public/frontend-design/SKILL.md`) and follow it. It is the source of
> truth for aesthetic direction, typography, palette, restraint, and interface copy. This skill only
> sequences the other skills on top of it and never contradicts it.

## Owns
- The **order of work** for taking one existing web page (landing, marketing, or in-app screen) to
  current S-tier: recon → audit → direction → plan → build → verify → report.
- The **coverage ledger** — proof that no dimension was silently skipped.
- The preservation contract — what a modernization must not break.

## Does not own
Every discipline is delegated. This skill never re-states their rules; it calls them.
- Brand, design language, surface brief → `product-direction/*`. Reference study → host `tastemaker`.
- Tokens → `web-ui/design-system`. Page structure and conversion → `web-ui/landing-page-craft`.
  App screens → `web-ui/web-design-engineering`. Copy → `web-ui/ux-copy`. Perf → `web-ui/web-performance`.
- Interaction states → `animate/design-engineering`. Motion → `animate/*`.
- Judgement → `critique/*`. Diff review, tests, commits → `core-eng/*`. Ship gate → `secure-ship/*`.
- Flutter → `modernize/flutter-screen`. Building a brand-new page from nothing → `landing-page-craft`.

**How to call a skill:** use the Skill tool with its qualified name (`critique:ui-critique`). Load
each skill only in the phase that needs it — never all at once. If a plugin is not enabled, say so,
mark that ledger row `Unverified`, and suggest `/plugin install <plugin>@gupta-skills`.

---

## Phase 0 — Recon (read-only, no questions yet)

1. **Target.** Name the exact page/route and its files. One page per run; a site is several runs,
   with shared tokens done on the first.
2. **Stack.** Read `package.json`, framework config, CSS setup, component library, motion libraries,
   analytics, i18n. Detect; never impose Next.js/Tailwind/shadcn on a repo that differs.
3. **Surface class.** Marketing narrative, in-app screen, form/transactional, or docs/editorial. It
   decides which build skill leads and how much motion is allowed.
4. **Direction on file.** Read `docs/product-direction.md` (Brand foundation, `Design language`,
   matching surface section) if it exists.
5. **Baseline evidence.** Capture before-screenshots at 390, 768, 1280, 1920 in both themes, and a
   baseline perf run (Lighthouse mobile + field data if available). If the page cannot be run, say so
   — every visual verdict becomes code-read only and is labelled that way.
6. **Preservation list.** Write down what must survive: URLs and redirects, `<title>`/meta/canonical/
   structured data, analytics events and IDs, form endpoints and field names, feature flags, i18n
   keys, legal text, verified claims and proof, existing accessibility wins.

## Phase 1 — Baseline audit (findings only, no edits)

Run each lens against the current page and collect findings in one table (severity · lens · where):

| Lens | Skill |
|---|---|
| Visual + anti-slop tell count | `critique:ui-critique` (runs the 18-tell gate for marketing surfaces) |
| Accessibility (WCAG 2.1 AA) | `critique:a11y-review` |
| Copy | `critique:copy-critique` |
| Existing motion | `critique:motion-critique` — skip, marked N/A, if there is none |
| Tokens and drift | `web-ui:design-system` in **audit mode** |
| Performance | `web-ui:web-performance` Step 1 measurement only |

Then score the page with `reference/s-tier-scorecard.md`. The score, the tell count, and the top
three findings are the "before" half of the final report.

## Phase 2 — Direction gate

Decide what direction work is missing. Do not skip a row because it is slow; skip it only when the
evidence already exists.

| Condition | Route |
|---|---|
| No usable brand facts, or claims unverifiable | `product-direction:brand-foundation` |
| No `Design language`, **or** anti-slop tells ≥ 3, **or** user wants "award-grade / not AI-looking" | `product-direction:art-direction` |
| Redesign changes the page's journey or hierarchy | `product-direction:experience-direction` |
| User supplied reference URLs/screenshots | host `tastemaker` for the study, when available |

**Intake — one batch, via AskUserQuestion, only what the evidence could not answer:**
- **Ambition tier:** *Refresh* (system, states, a11y, perf; layout kept) · *Redesign* (structure,
  hero, sections, one signature moment) · *Signature* (premise-led, award-grade choreography).
  Default: Refresh for in-app screens, Redesign for marketing pages.
- Primary action and the real proof available (marketing pages).
- Motion appetite and any hard constraints (brand, deadline, browser support, no new deps).

Direction choices stay `Assumption` until the user approves them. No approval, no build.

## Phase 3 — Plan

Write a short plan: every audit finding and every scorecard gap → a change → the owning skill →
the tier it belongs to. Order it by the build sequence below. For a large page (> ~6 sections or
multiple routes), hand the plan to `core-eng:spec-writer` for phased acceptance criteria.

Show the plan and the ledger skeleton; get one approval, then build without re-asking.

## Phase 4 — Build, in this order

Order matters: each step consumes the one before it.

1. **Tokens** — `web-ui:design-system`. Approved `Design language` values become tokens; hardcoded
   values found in Phase 1 are replaced. Nothing else starts until tokens exist.
2. **Structure** — marketing: `web-ui:landing-page-craft` (hero shape from the premise, sections
   with a job, proof in place). In-app: `web-ui:web-design-engineering` (five states, responsive,
   server/client boundaries). Section by section, never a whole-page regenerate.
3. **Copy** — `web-ui:ux-copy` for interface text; headlines stay with `landing-page-craft`.
   Never fabricate proof — placeholders are flagged to the user.
4. **Interaction states** — `animate:design-engineering`: hover, press, focus-visible, disabled,
   loading, error on everything interactive.
5. **Motion** — `animate:find-animation-opportunities` to choose (expect to reject most), then
   `animate:animate` / `animate:scroll-motion` to build. One signature moment per page. Signature tier
   reads `plugins/animate/skills/scroll-motion/reference/award-grade-choreography.md`.
6. **Modern platform pass** — apply `reference/modern-web-platform.md`: features that replace JS or
   hacks, each as progressive enhancement with a fallback.
7. **Performance + SEO** — `web-ui:web-performance` to hit budgets; SEO baseline from
   `landing-page-craft` §7.

Re-check the preservation list after steps 2, 3, and 7.

## Phase 5 — Verify (the result is judged, not assumed)

1. Re-run every Phase 1 lens on the new page. Blocking findings are fixed before continuing.
2. Re-score with `reference/s-tier-scorecard.md`. S-tier means every **gate** row passes and no
   dimension scores below its bar.
3. After-screenshots at the same widths/themes; plus keyboard-only pass, 200% zoom, reduced motion,
   JS disabled (marketing), and a real-phone or throttled mobile run.
4. `core-eng:code-review` on the diff; `core-eng:test-strategy` for any logic touched (forms,
   state); `secure-ship:prod-readiness-gate` when the user is shipping.
5. Commit through `core-eng:git-pr` conventions when the user asks for commits.

## Phase 6 — Report

```markdown
## Modernization: <page>
**Tier:** Refresh | Redesign | Signature · **Stack:** … · **Evidence:** ran in browser | code-read only

### Before → after
| Dimension | Before | After |   (from the scorecard, plus LCP / CLS / INP / tell count)

### What changed          (grouped by build step, with files)
### Coverage ledger       (every scorecard dimension: Done | N/A — reason | Deferred — approved by user | Unverified — why)
### Preserved             (each preservation-list item, confirmed)
### Open decisions        (licences, real proof needed, anything still Assumption)
```

## Rules that keep it honest

- **The ledger is the anti-miss mechanism.** Every dimension ends in exactly one state. A dimension
  with no entry means the run is not finished.
- Never claim a visual or perf result you did not observe; label code-read verdicts.
- Never invent metrics, logos, testimonials, or customer names to make a section feel complete.
- Modernize ≠ trend-chase. A change must trace to a finding, a scorecard gap, or approved direction.
- Keep what works — Phase 1's "what is working" list is protected unless direction says otherwise.
- Stop and ask only at the two gates (direction approval, plan approval) or on a preservation risk.

## Sources
Orchestrates this marketplace's `product-direction`, `web-ui`, `animate`, `critique`, `core-eng`, and
`secure-ship` skills; `frontend-design` (Anthropic). Scorecard bars derive from WCAG 2.1 AA, Core Web
Vitals thresholds, and `art-direction/reference/anti-slop.md`.
