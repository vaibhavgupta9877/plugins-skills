---
name: ui-critique
description: Review a UI screen, mockup, or component for hierarchy, spacing, typography, consistency, and whether it reads as templated. Do NOT use to review motion — that's motion-critique.
---

# ui-critique

> **Read first.** Invoke the `frontend-design` skill (plugin `frontend-design@claude-plugins-official`;
> on Linux sandboxes, `/mnt/skills/public/frontend-design/SKILL.md`) and follow it. It is the source of
> truth for aesthetic direction, typography, palette, restraint, and interface copy. This skill judges
> against that standard and never contradicts it.

## Owns
- Visual design review: hierarchy, layout, spacing rhythm, type, colour.
- The "does this look AI-generated" judgement.
- Cross-screen consistency of the same component.

## Does not own
- Motion quality → `motion-critique`.
- Accessibility conformance → `a11y-review`.
- Interface wording → `copy-critique`.
- Building the fix → `web-ui/*`, `flutter/flutter-ui-craft`.

**This skill produces findings. It does not write production code.** If the user then says "fix it,"
control passes to the relevant building skill.

When `docs/product-direction.md` exists, use its verified Brand foundation, `Design language`, and matching surface section when present; check execution against the language's refusals, type, colour, and motion signature. If no surface section exists, review the actual screen and known brief, state any resulting scope limit, and do not infer a direction violation from a missing section. Keep assumptions distinct from facts; palette/type resemblance alone is not automatically wrong.

---

## Output contract (shared by all critique skills)

Findings ranked by severity — `blocking` / `should-fix` / `nit`. Each finding states **what · where ·
why it matters · concrete fix**. End with **the single most important change**.

Never pad with manufactured findings. "This is solid, here are two nits" is a valid and valuable
output. A review that invents problems to look thorough trains the user to ignore reviews.

---

## Step 0 — Say what is working

Two or three sentences, specific, before any criticism. Not flattery: the parts you name as working
are the parts the user must not break while fixing the rest. Reviews that only attack get ignored.

---

## Step 1 — The templated-default check

`frontend-design` names three looks that AI-generated design currently converges on:

1. Warm cream ground (near `#F4F1EA`) + high-contrast serif display + terracotta accent.
2. Near-black ground + a single acid-green or vermilion accent.
3. Broadsheet layout — hairline rules, zero radius, dense newspaper columns.

Treat these as diagnostic prompts, not a palette blacklist. First check the brief and approved
`docs/product-direction.md` when present. **A resemblance alone is never a `blocking` finding.**
When no product-specific rationale is evidenced and the pattern plausibly dilutes identity, report
`should-fix` and give one concrete grounding or differentiation suggestion. Raise to `blocking` only
when the pattern materially contradicts verified direction or demonstrably harms hierarchy,
readability, trust, accessibility, or performance. If approved direction specifies the style and the
execution fits, report no finding for that style.

Adjacent tells worth checking with the same evidence standard: purple-to-pink-to-blue "tech" gradient,
rounded card with a coloured left-border accent, emoji standing in for icons, AI-drawn SVG humans,
Inter/Roboto/system-ui carrying the display role, cyber-neon on `#0D1117`, fabricated stats, and fake
logo walls. Fabricated proof or inaccessible UI may be `blocking` because of its direct harm, not
because it resembles a style pattern.

Then ask the harder version: **would this same design have come out for a different brief in the same
category?** If yes, report `should-fix` when no product-specific rationale is evidenced, and name a
subject-specific material, instrument, artifact, or vernacular that could ground it. A generic-feeling
direction alone is not `blocking` unless it also causes a material task, trust, or accessibility harm.
Do not infer competing signatures or add token/documentation nits without visible evidence. If the
honest result is no finding, say so and stop.

For marketing, portfolio, or "make it premium / not AI-looking" reviews, also run the 18-tell gate and
scoring in `plugins/product-direction/skills/art-direction/reference/anti-slop.md` (count tells; 3+
means the design language is underspecified — say so as one finding and route to `art-direction`
rather than listing pixel fixes). The same evidence standard applies.

---

## Step 2 — Check order

Work these in order. Never lead with colour or radius; a hierarchy problem invalidates everything
downstream of it.

1. **Hierarchy vs priority.** Rank what the content actually is, most to least important. Now rank
   what the design emphasises. Where the two lists disagree, that is the finding. The eye should land
   on the primary thing first, and you should be able to say why it does.
2. **Spacing scale.** Is every gap on one scale, or are there arbitrary values? Inconsistent spacing
   is the most visible symptom of a missing system. Judge rhythm *between* sections separately from
   rhythm *inside* components — they are different problems with different fixes.
3. **Type scale.** Deliberate — a ratio, named roles, intentional weights and widths — or a pile of
   sizes? Is the type treatment itself memorable, or a neutral delivery vehicle for the content?
4. **Colour doing work.** Every colour should carry meaning (state, hierarchy, brand) or be
   structural. Colour that only decorates is a finding. Check semantic colours actually read as their
   meaning, in both themes.
5. **The signature element.** Is there one? Is there exactly one? "Spend your boldness in one place"
   means one memorable element with everything around it quiet. Two competing signatures is the same
   defect as none.

---

## Step 3 — Consistency sweep

**The same component with different spacing, radius, shadow, or type across screens is the most
common real defect in shipped product UI.** It is also the easiest to prove and the cheapest to fix,
so it belongs in every review.

Sweep for:
- One component, several implementations — three button paddings, two card radii.
- Hardcoded values where a token exists. Grep for hex colours and raw px/dp in component files.
- Icon sets mixed, or icon weights mixed within one set.
- Radius drifting between nested shapes. Where a shape sits inside another and the gap is under
  ~32px: `inner radius = outer radius − gap`, applied only when the result exceeds 2.
- Elevation used inconsistently — shadow meaning "raised" here and "selected" there.
- Dark mode treated as an inversion rather than a designed second theme.

---

## Step 4 — Composition and detail

- Alignment: is there a grid, and does everything sit on it or deliberately break it? Near-misses
  read as sloppy far more strongly than intentional breaks read as bold.
- Density: is whitespace being spent evenly everywhere instead of used to group?
- Gestalt grouping: do related things look related through proximity, not only through a border?
- Optical vs mathematical centring on icons, glyphs, and circular shapes.
- Text measure: 45–75 characters for body copy. Long lines are a readability finding.
- Orphans and awkward heading breaks. `text-wrap: balance` on headings, `pretty` on body copy.
- Structural devices (numbering, eyebrows, dividers) must encode something true about the content.
  `01 / 02 / 03` is correct only when the content genuinely is a sequence.
- Empty, loading, and error states. If you can only see the populated state, say the review is
  partial and name what you could not judge.

---

## Step 5 — What you are reviewing

Adjust what you can honestly claim to the artefact in front of you.

- **Screenshot or mockup** — judge freely, but you cannot see states, motion, or responsive
  behaviour. Say so rather than implying full coverage.
- **Live URL or running app** — resize it, tab through it, and trigger the states before judging.
- **Figma description** — judge the described system; ask for the frames you cannot see.
- **Flutter widget tree** — read `ThemeData` / `ThemeExtension` first, then look for hardcoded
  `Color(0xFF…)` and `EdgeInsets` literals in widget files.
- **Web code** — read the token layer first, then grep components for values that bypass it.

---

## Step 6 — Report

Use the output contract. Anchor every finding to a screen name, a `file:line`, or a described region:
"the pricing card's header" is anchored, "the spacing" is not.

Close with the single most important change and what it costs. If the honest answer is that the
design is good, say that, name what makes it good, and stop.

## Sources
`frontend-design` (Anthropic); designer-skills `visual-critique`; jakubkrehel `interface-review`;
ConardLi `web-design-engineer` (AI-cliché taxonomy).
