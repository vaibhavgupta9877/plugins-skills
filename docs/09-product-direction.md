# Plugin: `product-direction`

**Enable for:** product UI/experience work across web, SaaS, Flutter, and related surfaces when brand context is missing or needs a design-ready update, and for substantial surface redesigns.
**Purpose:** a shared, evidence-grounded brand/product context and surface-direction handoff. It does not build or critique UI.
**Skills:** 2 Supporting.

---

## Ownership

- `brand-foundation` records concise design-relevant product and brand context, distinguishing verified facts, assumptions, and unknowns.
- `experience-direction` translates that context into an experience brief for one surface or flow.
- `web-ui`, `flutter`, and `animate` implement the handoff; `critique` reviews the result.
- `tastemaker` owns URL/screenshot reference study. This plugin consumes approved findings and does not create a competing study workflow.
- `product-marketing` and `customer-research` own full positioning, campaign strategy, and research execution; this plugin reuses their approved context rather than replacing it.

## `brand-foundation` — SUPPORTING

**Description:** `Use when a product UI/experience-design task needs an evidence-sourced brand foundation or update. Do NOT use for full product positioning/GTM or market research (product-marketing/customer-research); reuse those outputs here.`

**Owns:** evidence-grounded product and brand context needed by design work.
**Does not own:** market research, campaign strategy, surface experience briefs, visual styling, implementation, or critique.

**Required behavior:**
- Read approved product/brand/surface material before asking questions; don't repeat answered intake.
- Treat a product category as context, not evidence for specific features, customer outcomes, market position, or company scale.
- Classify material statements `Verified`, `Assumption`, or `Unknown`, with sources/rationale.
- Create/update only the Brand foundation section of the consuming project's `docs/product-direction.md`.
- Use the exact seven-section artifact tree and claim-handling rules in `plugins/product-direction/skills/brand-foundation/SKILL.md`.

**Budget:** 100 lines maximum.

## `experience-direction` — SUPPORTING

**Description:** `Use when translating approved product/brand context into direction for a new or redesigned product surface/flow. Do NOT use for site architecture (site-architecture), reference study (Tastemaker), or UI implementation/review (web-ui/*, flutter/*, animate/*, critique/*).`

**Owns:** surface-specific experience direction, acceptance criteria, and handoff.
**Does not own:** product/brand foundation, reference study, UI/motion implementation, or critique.

**Required behavior:**
- Read only the matching product/brand and surface context from `docs/product-direction.md` when it exists; otherwise use relevant facts from the current user brief and approved project docs.
- Preserve evidence granularity: do not infer roles, features, fields, metrics, or permissions from broad audience/category labels.
- Ground the surface type, user intent, journey, hierarchy, visual direction, interaction, accessibility/performance, and acceptance criteria in evidence; label hypotheses and unknowns.
- Numeric acceptance targets require an approved product target or named standard; otherwise keep them as assumptions to validate, not requirements.
- Use a product-grounded visual idea only when one is earned. A reference can inform principles, never license copying its identity, copy, imagery, or exact interaction sequence.
- For raw URL/screenshot references, hand off to Tastemaker. Read `plugins/product-direction/skills/experience-direction/reference/case-study-principles.md` only when curated examples are useful.
- Produce the exact surface brief structure in `plugins/product-direction/skills/experience-direction/SKILL.md`; do not write UI or review a finished screen.

**[R4] First instruction:** the skill body begins with the exact `frontend-design` directive required by `docs/00-INDEX.md`. That skill remains the aesthetic authority.

**Budget:** 120 lines maximum.

## Shared artifact

`docs/product-direction.md` belongs to the consuming product repository, not this plugin repository. It contains a Brand foundation section and a separate section for each surface. Preserve approved existing content; update only the relevant section. Every material statement keeps its source/status; no assumption is promoted to a public claim without evidence and user approval.

## Sources

- `frontend-design` (Anthropic) supplies the aesthetic authority; do not duplicate its rules.
- Tastemaker supplies reference-study findings when a URL or screenshot is provided.
- The curated site analyses and access caveats are in `plugins/product-direction/skills/experience-direction/reference/case-study-principles.md`.
