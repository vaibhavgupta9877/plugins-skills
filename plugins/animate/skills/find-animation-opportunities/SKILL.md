---
name: find-animation-opportunities
description: Scan an existing static UI and identify where motion would add real value, ranked by payoff. Do NOT use to implement the motion — that's animate.
---

# find-animation-opportunities

> **Read first.** Invoke the `frontend-design` skill (plugin `frontend-design@claude-plugins-official`;
> on Linux sandboxes, `/mnt/skills/public/frontend-design/SKILL.md`) and follow its restraint guidance.
> Extra animation makes a design feel AI-generated. **This skill is a filter as much as a finder.**

## Owns
- Discovering and ranking places where motion would earn its cost.

## Does not own
- Implementation → `animate`.
- Improving motion that already exists → `improve-animations`.
- Judging existing motion → `critique/motion-critique`.

**Read-only. Propose, do not implement.** Expect to reject most candidates — an opportunity finder
that suggests motion everywhere produces exactly the over-animated interfaces this plugin exists to
prevent.

---

## Step 1 — Recon

Before judging anything: the stack, the motion libraries present, the existing easing and duration
tokens (suggestions must extend these, never invent a parallel set), and the product's personality.
A crisp dashboard earns fewer and subtler suggestions than a playful consumer app.

Then build a rough **frequency map** of the surfaces you are about to judge. It drives every verdict.

---

## Step 2 — Hunt only these four categories

Everything outside them is decoration hunting. Sweep for:

1. **State changes with no visual continuity** — content that swaps, appears, or vanishes instantly.
   Conditional renders, route content, expanding sections, list items added or removed, accordions
   that snap. Grep for `{isOpen &&`, `display: none` toggles, `.map(` over entering lists.
2. **Actions with no feedback** — a press that appears to do nothing. Pressable elements with no
   `:active` state; destructive actions confirmed by a plain click where a hold-to-confirm would
   prevent slips. Grep for `onClick` handlers on elements with no `:active` or transition styles.
3. **Spatial relationships asserted but not shown** — a panel, popover, or menu that appears with no
   connection to its trigger; a dismissable surface that exits by a different path than it entered.
4. **Loading and empty states that read as broken** — a blank region during a fetch, a bare "no
   results", a first-run screen rendered flat. This is where the delight budget is best spent.

---

## Step 3 — Gate every candidate

All four questions, in order. Record the answers; they go in the report.

**1. Frequency**

| Seen | Verdict |
|---|---|
| 100+/day — shortcuts, command palette, core nav | **Reject. Never animate.** |
| Tens/day — hover, list nav, frequent toggles | Reject, or near-imperceptible only |
| Occasional — modals, drawers, toasts, settings | Eligible |
| Rare / first-time — onboarding, empty, success | Eligible; delight budget lives here |

Keyboard-initiated actions are a disqualifier, not a judgement call.

**2. Purpose** — name it: feedback · spatial consistency · state indication · preventing a jarring
change · explanation · delight (rare tier only). "It looks cool" is not on the list.

**3. Speed** — can it live inside budget? Press 100–160ms, tooltip 125–200ms, dropdown 150–250ms,
modal 200–500ms. If it only works as a slow showy animation, it fails.

**4. Function** — does motion help or hinder? Data the user is reading or acting on must not move for
style.

---

## Step 4 — Rank, and say how many to build

Order by leverage, not by how fun each would be to build.

**Cap the output at 5–7 suggestions for a whole app, fewer for a single view. Then state explicitly
how many to actually implement — usually 2–3 per screen.** A list of twelve opportunities is a
failure of judgement, not thoroughness.

---

## Step 5 — Report

**Part 1 — Opportunities**

| # | Location | Today | Purpose | Frequency | Suggested motion |
|---|---|---|---|---|---|
| 1 | `Toast.tsx:41` | Appears instantly | Preventing a jarring change | Occasional | `@starting-style`: `opacity:0; translateY(100%)` → settled, 250ms `--ease-out`; exits the same edge at 150ms |
| 2 | `Button.tsx:18` | No press feedback | Feedback | Tens/day | `:active { transform: scale(0.97) }`, `transition: transform 160ms ease-out` — subtle enough for the tier |

Every "suggested motion" cell carries **exact values** — properties, curve, duration — taken from the
repo's tokens or from `animate/reference/timing-table.md`, never approximated. `transform` and
`opacity` only. Include reduced-motion handling, and `@media (hover: hover) and (pointer: fine)`
gating on anything hover-driven.

**Part 2 — Opportunities deliberately not taken (REQUIRED)**

2–5 places you considered and rejected, each with the gate question that killed it:

- `CommandMenu.tsx:12` — open/close. **Rejected: keyboard-initiated, 100+/day. Never animate.**
- `Chart.tsx:88` — animated line draw. **Rejected: functional data being read; motion hinders.**

**This section is what separates this skill from an animation wishlist. Never omit it.**

**Part 3 — Verdict**

One short paragraph: how much motion this interface actually needs, whether it is already close to
right, and which single suggestion has the highest leverage. If nothing survives the gate, say so
plainly — that is a good result, not a failure. Hand implementation to `animate`.

## Sources
`frontend-design` (Anthropic); Emil Kowalski, "You Don't Need Animations".
