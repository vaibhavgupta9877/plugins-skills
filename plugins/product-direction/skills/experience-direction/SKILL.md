---
name: experience-direction
description: Use when translating approved product/brand context into direction for a new or redesigned product surface/flow. Do NOT use for site architecture (site-architecture), reference study (Tastemaker), or UI implementation/review (web-ui/*, flutter/*, animate/*, critique/*).
---

# experience-direction

> Before doing anything else, read `/mnt/skills/public/frontend-design/SKILL.md` and follow it. It is the source of truth for aesthetic direction, typography, palette, restraint, and interface copy. This skill adds stack-specific execution on top of it and never contradicts it.

## Owns
- Translating approved product/brand context into a surface-specific experience direction and acceptance criteria.

## Does not own
- Product/brand foundation → `brand-foundation`.
- Multi-page site architecture/navigation → optional host skill `site-architecture`, when available.
- URL/screenshot reference study → optional host skill `tastemaker`, when available.
- UI, motion implementation, or critique → `web-ui/*`, `flutter/*`, `animate/*`, `critique/*`.

## Core principle

Every material design recommendation traces to an evidenced user task, product truth, explicit user preference, or approved reference finding. A reference's visual identity is not the product's identity.

## Inputs and workflow

1. Read `docs/product-direction.md` when it exists and use only the matching brand/surface sections. If foundational context is missing, route to `brand-foundation`; do not invent it here.
2. Classify the surface (marketing narrative, app shell/data view, transactional form, editor, or state screen) and record its actual use frequency and primary user task.
3. Build the direction around the task and evidence. Choose a visual idea only when it follows from product truth; explicitly choose no motif when none is earned. Do not invent product features, data, roles, metrics, or screens to make a direction feel complete.
4. For a raw URL or screenshot, hand it to Tastemaker for study when that host skill is available and consume its approved findings. If unavailable, request an approved report or screenshot and mark uninspected observations Unknown; use only user-stated preferences in the meantime. Do not claim direct inspection. Transfer only principles that fit the product task; never copy identity, copy, assets, or exact sequences. Read `reference/case-study-principles.md` when its curated examples are relevant.
5. Recommend interaction or motion only when it helps the stated task. Frequent-use screens default to immediate, scannable states; any exceptional motion needs a named purpose and accessible static/reduced-motion behavior.
6. Update only the matching surface section in `docs/product-direction.md`. Ask only high-impact questions; label unresolved decisions rather than blocking useful provisional guidance.
7. Hand implementation to the relevant existing web, Flutter, or motion skill. This skill does not write UI or audit the result.

## Status contract

Use only `Verified`, `Assumption`, or `Unknown` for material claims. Every material bullet begins with exactly one status and includes a source or rationale. `Verified` means direct user/project evidence or an approved reference finding; `Assumption` means a clearly stated design hypothesis; `Unknown` names the evidence needed to resolve it. Preserve evidence granularity: “staff” does not verify receptionist/clinician/owner roles, and a voice preference does not establish palette or type. A proposed layout, control, motion choice, or data hierarchy not supplied by a source is an `Assumption`, not an existing product fact. Reference details are `Unknown` without an approved study.

## Required surface brief

For every request to create surface direction, an experience plan, or UI guidance—including casual requests like “give me a quick UI direction”—the final answer must consist only of this one structured surface section. Start with `## Surface: <name>`. Include every `###` heading exactly once and in order. Do not add an introduction, freeform proposal, alternate headings such as `Phone` or `Windows`, screen-by-screen appendix, or other headings. Put platform differences in bullets under the relevant headings.

```markdown
## Surface: <name>

### Type and frequency
### Audience and intent
### Primary journey
### Information hierarchy
### Product-grounded visual direction
### Evidence and assets
### Interaction and motion
### Accessibility and performance
### Acceptance criteria
### Assumptions and open questions
```

Keep the section at or below 500 words. Every material bullet begins with one allowed status and includes its source/rationale. Every bullet under `### Assumptions and open questions` begins `**Unknown** —` and names the source needed. If a field is absent, keep its heading and state `Unknown` rather than omitting it. Any numeric target (latency, taps, touch size, contrast, or motion timing) must cite an approved product target or named standard; otherwise mark it as an assumption to validate, not an acceptance requirement. Recommendations may be concrete, but mark them Assumption unless the source verifies them. This section is the whole handoff, not implementation code.

## Quick reference

| Surface signal | Direction |
|---|---|
| Marketing narrative | Carry a product-truth-led story; one visual idea can earn a larger moment. |
| Daily task UI | Prioritize the primary task, scanability, keyboard/touch paths, and instant usable states. |
| Missing brand context | Use `brand-foundation`; label unknowns meanwhile. |
| Raw URL/screenshot reference | Use `tastemaker` for study, then adapt only relevant principles. |

## Example

```markdown
### Audience and intent
- **Verified** — The audience is clinic staff. Source: brief.
- **Unknown** — Whether this includes front-desk staff, clinicians, or owners. Source needed: role research or product spec.
```

## Common mistakes

- Treating a reference's requested style as evidence of the product's identity.
- Copying a metaphor, typography, copy, assets, or interaction sequence wholesale.
- Inventing screens, data fields, metrics, or workflows to make a recommendation feel complete.
- Applying cinematic motion to a high-frequency task surface without a purpose or accessible instant state.
- Producing recommendations without a stable source-labelled handoff.

## Sources

`frontend-design` (Anthropic); `tastemaker` for URL/screenshot study; `reference/case-study-principles.md` for the plugin's curated case-study notes.
