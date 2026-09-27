---
name: experience-direction
description: Use when translating approved product/brand context into direction for a new or substantially redesigned product surface or flow. Do NOT use for reference study (Tastemaker) or UI implementation/review (web-ui/*, flutter/*, animate/*, critique/*).
---

# experience-direction

> Before doing anything else, read `/mnt/skills/public/frontend-design/SKILL.md` and follow it. It is the source of truth for aesthetic direction, typography, palette, restraint, and interface copy. This skill adds stack-specific execution on top of it and never contradicts it.

## Owns
- Translating approved product/brand context into a surface-specific experience direction and acceptance criteria.

## Does not own
- Product/brand foundation → `brand-foundation`.
- URL/screenshot reference study → `tastemaker`.
- UI, motion implementation, or critique → `web-ui/*`, `flutter/*`, `animate/*`, `critique/*`.

## Core principle

Every material design recommendation traces to an evidenced user task, product truth, explicit user preference, or approved reference finding. A reference's visual identity is not the product's identity.

## Inputs and workflow

1. Read `docs/product-direction.md` when it exists and use only the matching brand/surface sections. If foundational context is missing, route to `brand-foundation`; do not invent it here.
2. Classify the surface (marketing narrative, app shell/data view, transactional form, editor, or state screen) and record its actual use frequency and primary user task.
3. Build the direction around the task and evidence. Choose a visual idea only when it follows from product truth; explicitly choose no motif when none is earned. Do not invent product features, data, roles, metrics, or screens to make a direction feel complete.
4. For a URL or screenshot, consume an approved Tastemaker study. If none exists, hand off to Tastemaker; do not claim to have inspected visual details. A user request for a reference style is a verified preference, not proof of the reference's content. Transfer only principles that fit the product task; never copy identity, copy, assets, or exact sequences. Read `reference/case-study-principles.md` when its curated examples are relevant.
5. Recommend interaction or motion only when it helps the stated task. Frequent-use screens default to immediate, scannable states; any exceptional motion needs a named purpose and accessible static/reduced-motion behavior.
6. Update only the matching surface section in `docs/product-direction.md`. Ask only high-impact questions; label unresolved decisions rather than blocking useful provisional guidance.
7. Hand implementation to the relevant existing web, Flutter, or motion skill. This skill does not write UI or audit the result.

## Status contract

Use only `Verified`, `Assumption`, or `Unknown` for material claims. Every material bullet begins with exactly one status and includes a source or rationale. `Verified` means direct user/project evidence or an approved reference finding; `Assumption` means a clearly stated design hypothesis; `Unknown` names the evidence needed to resolve it.

## Required surface brief

Output exactly this Markdown tree, with the surface name supplied by the brief:

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

Add no other headings. Keep the surface section at or below 500 words. Every material bullet in all sections begins with one allowed status prefix and includes its source/rationale. Every bullet under `### Assumptions and open questions` begins `**Unknown** —` and names the source needed to resolve it. This section is the whole handoff, not implementation code.

## Quick reference

| Surface signal | Direction |
|---|---|
| Marketing narrative | Carry a product-truth-led story; one visual idea can earn a larger moment. |
| Daily task UI | Prioritize the primary task, scanability, keyboard/touch paths, and instant usable states. |
| Missing brand context | Use `brand-foundation`; label unknowns meanwhile. |
| Raw URL/screenshot reference | Use `tastemaker` for study, then adapt only relevant principles. |

## Example

```markdown
### Primary journey
- **Verified** — The user reviews transactions and approves expenses several times a day. Source: brief.

### Assumptions and open questions
- **Unknown** — Transaction fields and approval states. Source needed: product spec or current screen.
```

## Common mistakes

- Treating a reference's requested style as evidence of the product's identity.
- Copying a metaphor, typography, copy, assets, or interaction sequence wholesale.
- Inventing screens, data fields, metrics, or workflows to make a recommendation feel complete.
- Applying cinematic motion to a high-frequency task surface without a purpose or accessible instant state.
- Producing recommendations without a stable source-labelled handoff.

## Sources

`frontend-design` (Anthropic); `tastemaker` for URL/screenshot study; `reference/case-study-principles.md` for the plugin's curated case-study notes.
