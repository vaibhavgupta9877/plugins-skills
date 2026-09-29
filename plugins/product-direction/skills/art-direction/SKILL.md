---
name: art-direction
description: Use when a site or product needs a distinctive, award-grade design language, or its UI reads generic or AI-generated. Do NOT use for tokens (design-system), reference study (tastemaker), or page builds (web-ui/*).
---

# art-direction

> Before doing anything else, read `/mnt/skills/public/frontend-design/SKILL.md` (or invoke the `frontend-design` skill) and follow it. It is the source of truth for aesthetic direction, typography, palette, restraint, and interface copy. This skill adds a recorded, product-specific design language on top of it and never contradicts it.

## Owns
- The **design language** for a product or site: one premise, and the type, colour, layout, imagery, motion, and interaction grammar that express it.
- The anti-slop gate: rejecting defaults before they reach implementation.

## Does not own
- Brand facts and claims → `brand-foundation`. Per-surface journey and hierarchy → `experience-direction`.
- URL/screenshot study → host skill `tastemaker`, when available.
- Tokens → `web-ui/design-system`. Page build → `web-ui/landing-page-craft`, `web-ui/web-design-engineering`, `flutter/flutter-ui-craft`. Motion build → `animate/*`. Review → `critique/*`.

## Core principle

**Premium is coherence, not decoration.** Award-grade sites (Awwwards SOTD/SOTM, 2026) share one trait: every type, colour, grid, and motion choice serves a single idea native to the subject. AI slop is the opposite — safe defaults chosen independently. A design language is a set of *refusals* as much as choices.

## Workflow

1. **Read context.** `docs/product-direction.md` Brand foundation and matching surface sections, existing UI, and any approved reference findings. Missing brand facts → `brand-foundation`; do not invent them.
2. **Find the premise.** Write one sentence: *"This site behaves like ___ because ___."* The blank is an object, place, ritual, or material native to the subject (a chessboard for a finance novel; a staff roster for an AI-agents studio; a terrain for Gulf advisory). If nothing native exists, the premise is typographic — the words are the image. Never pick a premise because it looks impressive. Read `reference/archetypes.md` for grammars that fit common subjects; adapt, never skin-copy.
3. **Run the slop gate.** Read `reference/anti-slop.md`. List the three defaults this brief would attract (by category and audience) and name what replaces each.
4. **Define the language** using `reference/design-language-2026.md` for current patterns and concrete values:
   - **Type** — a display face with character + a workhorse (often mono or grotesk) for labels/data; a fluid scale with tracking that tightens as size grows. Name faces, or name the criteria and flag the licence decision.
   - **Colour** — a ground, an ink, one accent, and their dark/alt variants. Ground carries mood; the accent is rationed.
   - **Layout grammar** — grid, margins, the one rule the grid breaks, section indexing, label system, density rhythm.
   - **Imagery** — what is shown (real artifact, authored 3D, photography, none) and how it is treated. No stock, no generic AI imagery.
   - **Motion signature** — tempo (snappy / editorial / cinematic), 2–3 named easings, the one signature moment, page-transition stance, smooth-scroll stance, reduced-motion behaviour.
   - **Interaction signature** — cursor, hover language, one optional *toy* (drag, theme switch, physics) that expresses the premise.
   - **Voice hooks** — label and microcopy patterns that match the premise (coordinates, notation, indices).
5. **Budget the spectacle.** One signature moment per page; WebGL/3D only when it explains or atmospheres the subject and has a static fallback; a smooth-scroll engine only for narrative marketing surfaces. Frequent-use product UI inherits type, colour, and voice — not the theatre.
6. **Write the artifact** (below) into `docs/product-direction.md`, then hand off: tokens → `design-system`, pages → `landing-page-craft`, motion → `animate/scroll-motion` / `animate/animate`, review → `critique/ui-critique`.

## Required artifact

Create or update exactly this section in the consuming project's `docs/product-direction.md`:

```markdown
## Design language

### Premise
### Refusals
### Type
### Colour
### Layout grammar
### Imagery and assets
### Motion signature
### Interaction signature
### Voice hooks
### Budget and fallbacks
### Open decisions
```

Rules:
- Keep all eleven headings, in order. ≤ 450 words.
- Every material bullet starts `**Verified** —`, `**Assumption** —`, or `**Unknown** —` (same contract as `brand-foundation`). A proposed style choice is `Assumption` until the user approves it; approved choices become `Verified` with "Source: user approval, <date>".
- **Refusals** lists at least three specific defaults being rejected and their replacements.
- **Type** and **Colour** give concrete values (families, hex, clamp ranges) or name the decision in `Open decisions`.
- **Motion signature** names easing curves as values, the signature moment, and the reduced-motion and no-JS states.
- **Budget and fallbacks** states JS/3D weight stance, mobile simplification, and what degrades.
- Font licences (commercial foundry faces) and any imagery rights go in `Open decisions` unless verified.

## Quick reference

| Situation | Move |
|---|---|
| "Make it look premium / Awwwards" | Premise first. Premium without a premise becomes a trend collage. |
| Brief names a reference site | Transfer principles (tempo, grid discipline, pairing logic), never its identity, copy, assets, or sequence. |
| B2B / SaaS marketing | Type-led premise, one saturated or deep ground, mono labels, real product surfaces; 3D only if it explains. |
| Portfolio / studio | The work is the imagery; the grid and transitions are the craft proof. |
| Book / culture / launch | Let the artifact's native system (notation, chapters, format) drive layout and labels. |
| In-app product UI | Inherit type, colour, voice, easings. No smooth scroll, no preloader, no cinematic motion. |
| User says "it looks AI-generated" | Run `reference/anti-slop.md` against the current UI, then rewrite Refusals. |

## Common mistakes

- Picking a trend list instead of a premise — the result is a 2026 template instead of a 2023 one.
- Copying a reference's palette, font, or signature interaction and calling it direction.
- Choosing faces and hex values from the category ("fintech = navy + Inter").
- Specifying spectacle everywhere; award juries reward one directed idea at 60fps, not ten effects.
- Forgetting the reduced-motion, no-JS, and mid-range-Android versions of the language.
- Presenting the language as approved. It is a proposal until the user says so.

## Sources

`frontend-design` (Anthropic); source inspection of cloudstudio.es, moneyincheck.org, marcuslorenzet.com, boonglobal.io, oxigen.sa (2026-09-29); Awwwards 2026 winners and juror commentary; see `reference/design-language-2026.md` for the full source list and access caveats.
