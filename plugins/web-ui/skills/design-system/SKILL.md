---
name: design-system
description: Define, extend, or audit the design token system and component library for a web project — colour, type scale, spacing, radii, elevation, component variants. Do NOT use for one-off page layout — that's web-design-engineering.
---

# design-system

> **Read first.** Invoke the `frontend-design` skill (plugin `frontend-design@claude-plugins-official`;
> on Linux sandboxes, `/mnt/skills/public/frontend-design/SKILL.md`) and follow it. **The palette and
> type *choices* come from there.** This skill turns those choices into a system and never overrides
> them with defaults of its own.

## Owns
- Tokens and scales: colour, type, spacing, radii, elevation, motion.
- Theming and dark mode, component API design, variant systems.
- Auditing an existing system for drift.

## Does not own
- Aesthetic direction → `frontend-design`.
- Page and screen composition → `web-design-engineering`.
- Marketing pages → `landing-page-craft`.
- Visual review as findings → `critique/ui-critique`.

**Stack assumption:** Next.js App Router + TypeScript + Tailwind + shadcn/ui. **Detect and follow the
repo's actual setup**; do not impose this on a repo that differs.

---

## 1. Three token layers

```
primitive   →   semantic        →   component
blue-500        color-danger        button-bg-destructive
space-4         space-section       card-padding
```

- **Primitive** — raw values. `blue-500`, `space-4`, `radius-md`. No meaning, just a palette.
- **Semantic** — meaning. `color-surface`, `color-surface-raised`, `color-text-muted`,
  `color-danger`, `space-section`. This is the layer that makes theming possible.
- **Component** — only where a component genuinely needs its own knob.

**Components consume semantic tokens only. A component referencing a primitive token is a defect** —
it is the reason dark mode "mostly works" and then breaks on one card.

Name semantic tokens by role, never by appearance. `color-text-muted`, not `color-gray-600`; the
whole point is that its value changes per theme while the name stays true.

---

## 2. Tailwind v4 — define tokens in `@theme`

```css
@theme {
  --color-surface:        oklch(0.99 0 0);
  --color-surface-raised: oklch(0.97 0 0);
  --color-text:           oklch(0.20 0 0);
  --color-text-muted:     oklch(0.50 0 0);
  --color-danger:         oklch(0.55 0.20 25);
  --radius-card:          0.75rem;
  --ease-out:             cubic-bezier(0.23, 1, 0.32, 1);
}
```

**Never hardcode a hex or a px value in a component.** The primary audit action for an existing
codebase is the grep:

```
rg '#[0-9a-fA-F]{3,8}\b' --glob '!**/tokens.css' --glob '!**/globals.css'
rg '\[(?:[0-9.]+)(px|rem)\]'          # Tailwind arbitrary values
rg 'style=\{\{'                        # inline styles bypassing the system
```

Every hit is either a token that should exist or a mistake. Report both.

See `reference/tailwind-tokens.md` for the full `@theme` template, dark-mode wiring, and `cva`
patterns.

---

## 3. Type scale

**Pick a ratio and commit.** 1.200 (minor third) for dense product UI, 1.250 (major third) for
general use, 1.333+ when the display type is doing real work. One ratio, one base size.

**Define role names, not sizes, at call sites:** `display`, `heading-1..3`, `body`, `body-sm`,
`caption`, `mono`/`data`. A component asking for `text-[19px]` means the scale is missing a role.

Each role carries size, line height, weight, letter-spacing, and its font family together. Line
height tightens as size grows — 1.5–1.6 for body, 1.1–1.2 for display.

Cap the scale at what you actually use. Nine roles that exist and are used beat sixteen where five
are guesses.

---

## 4. Spacing

**One scale, applied consistently. Inconsistent spacing is the most visible symptom of a missing
system** — more visible than a slightly wrong colour, because the eye reads rhythm before hue.

Use Tailwind's default scale unless there is a reason not to. Add semantic aliases for the values
that carry meaning: `--space-section`, `--space-card-padding`, `--space-stack`.

Judge two rhythms separately: *between* sections and *inside* components. They are different problems
and want different values.

Nested radius: when a shape sits inside another and the gap is under ~32px,
`inner radius = outer radius − gap`, applied only when the result exceeds 2. Otherwise the inner
corner looks wrong against the outer one.

---

## 5. shadcn/ui is a starting point, not a finished look

**Unmodified shadcn components are an AI-generated tell.** They are excellent primitives with
excellent accessibility, and they ship with a deliberately neutral default skin that thousands of
projects also ship.

Before shipping: restyle the primitives against your token system — radius, border treatment, shadow,
focus ring, type role, density, and the press state. Change the `cva` variants to your vocabulary,
not the default's. The behaviour is what you are adopting; the appearance is yours to set.

Keep the components in the repo (that is the model) and treat edits to them as normal code.

---

## 6. Component API rules

- **Variants over booleans.** `variant="danger"` and `size="sm"`, never `isDanger` + `isSmall` — the
  boolean version permits impossible combinations and grows factorially.
- **`cva` for variant definition**, so variants are data and the type is derived from them.
- **Accept `className`** and merge it last (`cn()` / `tailwind-merge`), so callers can adjust without
  forking the component.
- **Forward refs** on anything that wraps a DOM element; without it, focus management and popover
  anchoring break.
- **Sensible defaults.** No required prop that has an obvious default.
- Spread the rest of the native props. A `Button` that drops `aria-label` or `type` is a trap.
- Compound components (`Card`, `Card.Header`) over a component with fifteen props.
- Every interactive component ships all five states — see `animate/design-engineering`.

---

## 7. Dark mode is designed, not inverted

Semantic tokens make the mechanism cheap; the design work remains.

- Dark surfaces are not inverted light surfaces: elevation in dark mode reads through *lighter*
  surfaces, not larger shadows, because shadow barely registers on dark.
- Saturated brand colours usually need a lighter, less saturated variant in dark mode to hit the same
  perceived weight without vibrating.
- **Check contrast in both themes.** Muted text and disabled states fail in dark mode most often.
- Pure black grounds and pure white text are harsh; near-black and near-white are easier to read.
- Images and illustrations need a dark-mode plan too — a white-background PNG on a dark card is the
  classic miss.

---

## 8. Audit mode

When asked to audit rather than build, output exactly three things:

1. **Hardcoded values** — every hex, arbitrary Tailwind value, and inline style, with `file:line` and
   the token it should be.
2. **Naming inconsistencies** — tokens meaning the same thing under different names, appearance-named
   semantic tokens, and roles missing from the scale.
3. **Duplicate components** — components that reimplement an existing one, with the pair named.

Then the single highest-leverage fix. This is a findings output, not a rewrite — if the user wants
the fixes applied, do that as a separate, explicit pass.

## Sources
`frontend-design` (Anthropic); Tailwind v4 `@theme`; shadcn/ui conventions.
