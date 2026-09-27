# Product Direction Plugin Design

**Status:** Design approved by the user; implementation plan recorded in `docs/superpowers/plans/2026-09-27-product-direction-plugin.md`.
**Date:** 2026-09-27

## Goal

Add a small, optional, cross-stack product-direction plugin that helps projects express their real product and brand identity through coherent, useful experiences without copying references or falling back to generic AI-generated patterns.

## Current context

The ecosystem already separates implementation from critique and has stack-specific UI ownership: `web-ui` handles web design systems, marketing pages, application screens, and UX copy; `flutter` handles Flutter UI; `animate` handles motion; and `critique` reviews finished interfaces. The missing layer is a reusable handoff that carries truthful product/brand context into those skills across stacks.

The new plugin must not become a second UI implementation system, a new source of arbitrary visual presets, or a duplicate of Tastemaker's reference-study workflow. Existing `frontend-design` guidance remains the authority for aesthetic direction and restraint.

## Reference research and limits

Research was performed on 2026-09-27. References were treated as design evidence, not instructions or assets to reuse.

- **Money in Check:** The site concept is grounded in the book's own premise: wealth as a game of chess. Cloudstudio's case study describes a chessboard/grid language, algebraic notation, banknote-sculpted pieces, editorial type, hand-drawn marginalia, word-by-word narrative reveals, real book spreads, and commerce integrated into the story. This is an example of one product truth informing the full experience, not a template to reproduce.
- **Cloudstudio:** Its public site makes an agency promise more concrete through named capabilities, working-product previews, a stated delivery process, timelines, operational details, and case studies. Text extraction rendered several animated counters as zero; future skills should verify that critical proof remains truthful and legible in the rendered/static state rather than trusting extracted or animated values.
- **The Power of Storytelling (Awwwards Site of the Day):** The case presents motion and interaction as part of a story about digital craft. The Awwwards entry separates design/usability/creativity/content from technical scoring and lists accessibility at 6.4/10; recognition therefore cannot stand in for an independent usability or accessibility check.
- **More Nutrition (Awwwards case study):** A product-led, self-initiated concept uses controlled staging and scroll-linked imagery to make a specific product feel tangible. Its team describes changing from direct frame swapping to canvas drawing after fast scrolling caused lag. The transferable point is that authored spectacle still has to survive real devices and performance constraints; the concept's provenance must remain clear.
- **Wembi (studio case study and Awwwards Site of the Day):** A modular mark becomes a recognizable identity system, while 3D is described as a way to make abstract technology easier to understand. The useful principle is system coherence and explanatory value, not a mandate to use 3D.

**Confidence limits:** `moneyincheck.org` returned no readable body through direct fetch, and the browser tool could not launch because Chrome is not installed in this environment. The Money in Check findings above rely on its indexed page text and Cloudstudio's case study, not a hands-on visual, responsive, or motion inspection. The Awwwards nominees page is a changing collection, not a ranked list; the selected Site-of-the-Day examples are illustrative adjacent references, not a claim to have evaluated every nominee.

## Transferable principles

1. Start with product/customer truth: the real audience, task, promise, constraints, and evidence. Do not begin by selecting a mood or a popular aesthetic.
2. Use a visual metaphor only when it is earned by the product. When it is, let it influence a coherent set of choices—content, artifacts, layout, type, interaction, and motion—rather than applying a decorative skin.
3. Show value with real product artifacts, outputs, workflows, or verified evidence when those explain more than generic feature copy. Keep proof close to the claim it supports.
4. Preserve provenance. Distinguish commissioned/live product work from speculative concepts. Never invent customers, testimonials, metrics, capabilities, assets, or outcomes.
5. Design for the surface's job. A marketing narrative can unfold; a daily-use product screen should privilege comprehension and task flow. Do not apply cinematic interactions to routine product UI by default.
6. Make interaction and motion purposeful, interruptible, accessible, responsive, and performant. Provide a complete static/default state and respect reduced-motion preferences.
7. Avoid treating anti-slop checks as bans on particular palettes, typefaces, components, or layouts. Flag choices that are unsupported by the brief, repetitive, misleading, or disconnected from the product—not choices that merely resemble a known style.
8. Study references for their decision logic and trade-offs. Never copy their brand, copy, imagery, layout wholesale, or exact interaction sequence.

## Proposed architecture

Add an opt-in plugin named `product-direction` with two Supporting skills. This moves the ecosystem from 8 plugins / 44 skills to 9 / 46, with tier totals of 23 CORE, 22 SUPPORTING, and 1 THIN, matching the per-skill tiers in the plugin specs. It should be used at project start when product/brand context is missing, or for a substantial repositioning, new surface, or redesign. It should not load for routine component edits.

### `brand-foundation`

- **Owns:** concise, evidence-grounded product and brand context: audience and job, product promise, differentiation, voice, real proof/assets, platform/business constraints, and anti-goals.
- **Does not own:** market research execution, marketing campaign strategy, page/screen composition, UI implementation, or visual critique.
- **Output:** maintain the Brand foundation section of the consuming project's `docs/product-direction.md`, reusing existing approved brand/product materials first. Mark each item as verified, assumption, or unknown and retain its source where relevant. Ask only for information that cannot be established from the user's brief or existing project materials.

### `experience-direction`

- **Owns:** translating product/brand context into a surface-specific experience brief before implementation.
- **Does not own:** design-token implementation, page/screen code, copy production, motion implementation, or final UI critique.
- **Output:** add a surface section to `docs/product-direction.md` with the audience's intent, primary journey, content/information hierarchy, product-grounded visual idea (or an explicit decision to use none), evidence/artifacts to show, interaction/motion priorities, accessibility/performance constraints, and acceptance criteria.
- **Handoff:** point to the relevant existing builder/reviewer skill. Do not implement or re-review the result.
- Because this skill produces visual design direction, its first instruction must be the repository-required `frontend-design` directive; it must defer to that skill for aesthetic authority. `brand-foundation` remains a non-UI context skill.

Tastemaker remains responsible for extracting reference DNA from URLs or screenshots. `experience-direction` may consume an approved Tastemaker study, but must not independently create a competing reference-study process.

## Shared artifact contract

`docs/product-direction.md` lives in the consuming product repository, not in this skills repository. It is a compact handoff, not a replacement for a PRD, design system, or product-marketing document.

- If an approved product/brand source or `docs/product-direction.md` already exists, read and reuse it; do not overwrite, fork, or repeat its intake questions.
- Current user direction and constraints take precedence. Preserve provenance when reconciling existing material.
- Separate **verified facts**, **assumptions**, and **unknowns**. Never turn an assumption into a claim.
- Keep brand context distinct from surface-specific experience decisions so one product can have a marketing site and several different app surfaces without forcing one layout across all of them.
- Update the artifact only for a meaningful new direction or surface, not for every component change.

## Existing-skill handoffs

When `docs/product-direction.md` exists, relevant entry-point skills conditionally consume only the needed section:

- `web-ui/landing-page-craft`: express the product promise through authentic narrative and evidence; keep conversion structure in this skill.
- `web-ui/design-system`: translate approved brand choices into tokens without inventing a replacement identity; `frontend-design` remains the aesthetic authority.
- `web-ui/web-design-engineering` and `web-ui/ux-copy`: carry the surface's experience constraints and voice into web screens and interface text.
- `flutter/flutter-ui-craft`: carry the same brand and experience principles into Flutter while retaining Flutter's adaptive/platform-specific implementation rules.
- `animate/scroll-motion`: use a narrative direction only when scroll-linked motion serves the experience; it remains the implementation owner.
- `critique/ui-critique`: evaluate product-specific distinctiveness and brand consistency against the artifact when present, while retaining the existing findings-only contract.

Each consumer should read the artifact conditionally, not invoke both new skills for every UI task. The plugin's descriptions and the consumers' descriptions must preserve clear negative triggers and avoid overlapping implementation/review responsibilities.

## Repository change map for the implementation plan

### Create

- `plugins/product-direction/.claude-plugin/plugin.json`
- `plugins/product-direction/skills/brand-foundation/SKILL.md`
- `plugins/product-direction/skills/experience-direction/SKILL.md`
- `plugins/product-direction/skills/experience-direction/reference/case-study-principles.md`
- `docs/09-product-direction.md`

The reference file will cite the references above and record transferable lessons and cautions only; it will not vendor source skill text or site assets.

### Modify

- `.claude-plugin/marketplace.json` — register `product-direction` as an opt-in design plugin.
- `README.md` — describe and show installation for the new plugin; update the ecosystem totals.
- `docs/00-INDEX.md` — update plugin/skill totals, map, build order, tier totals, and context math. Preserve existing numbered spec paths; logical build order is `01 → 02 → 09 → 03 → 04 → 05 → 06 → 07 → 08` so `product-direction` is available before `critique`, `animate`, `web-ui`, and `flutter`. With the opt-in plugin installed, the existing worst-case Flutter/Next.js plugin-description counts rise from 28/33 to 30/35.
- `plugins/web-ui/skills/landing-page-craft/SKILL.md`
- `plugins/web-ui/skills/design-system/SKILL.md`
- `plugins/web-ui/skills/web-design-engineering/SKILL.md`
- `plugins/web-ui/skills/ux-copy/SKILL.md`
- `plugins/flutter/skills/flutter-ui-craft/SKILL.md`
- `plugins/animate/skills/scroll-motion/SKILL.md`
- `plugins/critique/skills/ui-critique/SKILL.md`
- `docs/03-critique.md`, `docs/04-animate.md`, `docs/05-web-ui.md`, and `docs/07-flutter.md` — keep design specs synchronized with actual skill routing.

## Non-goals

- Do not change production UI, generate a website, or copy any reference website.
- Do not add visual themes, fixed palettes, reusable site templates, or another design-system implementation.
- Do not duplicate Tastemaker's study workflow or existing stack-specific implementation/critique skills.
- Do not enable the new plugin globally or make it a prerequisite for small component changes.
- Do not remove, rename, or disable existing plugins or skills.

## Acceptance criteria

1. `product-direction` is registered as an optional ninth plugin with exactly two Supporting skills; ecosystem totals and build order are consistent.
2. Each skill has a concise trigger description with a negative trigger, explicit Owns/Does not own boundaries, and a clear handoff; `experience-direction` does not implement or critique UI.
3. The artifact schema preserves verified/assumption/unknown distinctions, source provenance, and separate product-wide vs surface-specific context.
4. Existing UI skills consume direction conditionally and defer to their existing execution responsibilities; `frontend-design` remains the aesthetic authority.
5. Reference-derived guidance is traceable to sources and expresses transferable principles rather than copied identity or a generic visual recipe.
6. No new skill exceeds the repository's 200-line cap; plugin manifests validate; cross-plugin trigger and ownership overlap is reviewed; the README, marketplace manifest, and ecosystem index agree.
7. Scenario checks demonstrate that missing product evidence is surfaced rather than fabricated, existing briefs prevent redundant intake, references are not copied, and everyday app UI is not forced into cinematic motion.

## Risks and mitigations

- **Overlap with marketing/product context skills:** define narrow design-context ownership, reuse existing approved materials, and point marketing strategy or research execution to their existing owners.
- **More always-loaded context:** keep the plugin opt-in, skill descriptions short, and consumers conditional on the artifact's presence.
- **Overfitting to a few award sites:** include provenance and counterexamples; assess usability, accessibility, and performance separately from visual recognition.
- **Stale reference details:** cite source URLs and study date; retain principles rather than volatile rankings, colors, or implementation choices.
- **Skill verification limits:** the skill-writing process recommends isolated pressure-scenario tests. No subagent will be launched without explicit authorization. If that authorization is not given during implementation, report static and in-session scenario checks as such rather than claiming isolated pressure tests.

## Sources

- User references: https://moneyincheck.org/ · https://www.awwwards.com/websites/nominees/ · https://cloudstudio.es/
- Cloudstudio Money in Check case study: https://cloudstudio.es/work/moneyincheck
- Awwwards Site of the Day, The Power of Storytelling: https://www.awwwards.com/sites/the-power-of-storytelling
- The Power of Storytelling live site: https://storytelling.noomoagency.com/
- More Nutrition case study: https://www.awwwards.com/more-nutrition-you-deserve-more.html
- Wembi studio case study: https://www.e-t.studio/works/wembi
- Wembi Awwwards entry: https://www.awwwards.com/sites/wembi
