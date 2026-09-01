---
name: a11y-review
description: Audit a UI against WCAG 2.1 AA — contrast, keyboard navigation, focus order, semantics, touch targets, screen reader behaviour. Do NOT use for visual design quality — that's ui-critique.
---

# a11y-review

## Owns
- Accessibility conformance findings, web and Flutter.

## Does not own
- Aesthetics, hierarchy, spacing → `ui-critique`.
- Motion and reduced-motion judgement → `motion-critique`. **This skill defers on that overlap:**
  note it once and route it, do not double-report.
- Fixing → `web-ui/*`, `flutter/flutter-ui-craft`.

**Findings only. This skill does not change code.**

---

## Output contract

Findings ranked `blocking` / `should-fix` / `nit`, each with **what · where · why it matters ·
concrete fix**. End with the single most important change.

**Distinguish conformance failures from usability improvements.** A WCAG 2.1 AA failure is
`blocking` — it is a standard, and for many buyers a contractual one. A thing that would merely be
nicer for screen reader users is `should-fix`. Inflating the second into the first is how
accessibility reports get ignored.

Say what you tested with. An audit from reading code alone cannot claim screen reader behaviour —
mark those items UNVERIFIED rather than passing them.

---

## Check order

Work top to bottom. Earlier items block more users.

### 1. Contrast
- Body text and images of text: **4.5:1**.
- Large text (≥18.66px bold, or ≥24px): **3:1**.
- UI component boundaries, icons carrying meaning, focus indicators, chart series: **3:1**.
- Check **both themes**. Dark mode fails contrast far more often, especially on muted/secondary text
  and on disabled states that still need to be readable.
- Text over an image or gradient: check the worst pixel, not the average.
- Placeholder text as the only label fails twice — contrast and labelling.

### 2. Keyboard reachability and visible focus
- Every interactive element reachable by Tab. A `div` with an `onClick` and no `tabindex` is
  unreachable — `blocking`.
- **A visible focus indicator on every focusable element.** `outline: none` with no replacement is
  the single most common `blocking` finding in real audits.
- No keyboard trap: you can always Tab out of what you Tabbed into.
- Escape closes dialogs, popovers, and menus. Enter/Space activate. Arrow keys move within composite
  widgets (tabs, menus, listboxes).

### 3. Focus order and management
- Tab order matches visual order. A CSS `order` or `grid-area` reflow that desynchronises DOM order
  from visual order is a finding.
- Opening a modal moves focus into it and traps it there; closing restores focus to the trigger.
- Route change moves focus to the new page's heading or main landmark, and announces it.
- Content inserted above existing content must not steal or displace focus.

### 4. Semantics
- Real elements, not `div`s with handlers: `button`, `a[href]`, `input`, `nav`, `main`, `ul`/`li`.
  The native element brings role, keyboard behaviour, and focus for free.
- One `h1`; heading levels descend without skipping. Headings describe structure, not styling.
- Landmarks present (`header`, `nav`, `main`, `footer`) so a screen reader user can jump.
- ARIA only where a native element cannot do it. **Wrong ARIA is worse than none** — an
  `aria-label` that disagrees with the visible label breaks voice control.
- Lists marked up as lists; tables with real `th` and scope; no layout tables.

### 5. Labels and instructions
- Every input has a programmatically associated label. Placeholder is not a label.
- Required fields marked in text, not by colour or an asterisk alone.
- Error messages associated with their field (`aria-describedby`), and identify the field by name.
- Grouped controls (radios, checkbox sets) wrapped in `fieldset`/`legend` or an equivalent group role.

### 6. Touch targets and pointer
- **≥44×44 CSS px** (Flutter: ≥48dp) for anything tappable, with adequate spacing between adjacent
  targets. Expand the hit area rather than the visual size where the design needs a small control.
- No hover-only affordance: anything revealed on hover must also be reachable by keyboard and touch.
- Gestures have a single-pointer alternative — swipe-to-delete needs a button too.

### 7. Images and media
- Meaningful images have alt text that conveys the *information*, not the file name.
- Decorative images marked decorative (`alt=""`, or Flutter `ExcludeSemantics`).
- Icon-only buttons have an accessible name.
- Video has captions; audio has a transcript; nothing autoplays with sound.
- Information is never carried by colour alone — a red border needs an icon or text too.

### 8. Dynamic state
- Async results, toasts, validation summaries, and inline saves announce via a live region
  (`aria-live="polite"`, `assertive` only for genuine urgency).
- Loading state is announced, not just spun.
- Expanded/collapsed, selected, checked, and busy states exposed via the correct attribute
  (`aria-expanded`, `aria-selected`, `aria-current`), not implied visually.

### 9. Zoom and reflow
- 200% browser zoom with no horizontal scroll and no clipped content.
- 320px-wide viewport still usable (WCAG reflow).
- Respects OS text-size settings.

---

## Next.js specifics
- Focus management on App Router navigation — client-side routing does not move focus by default,
  and this is a real `blocking` failure that almost every Next.js app ships with.
- A skip link as the first focusable element, visible when focused.
- Modal and sheet primitives: verify the library actually traps and restores focus rather than
  assuming it (Radix and Base UI do; hand-rolled `div` overlays usually do not).
- Toasts and tRPC mutation states wired to `aria-live`. A mutation that only changes a colour is
  invisible to a screen reader user.
- `next/image` requires an explicit `alt`; an empty string must be a deliberate decorative choice.

## Flutter specifics
- `Semantics` on custom widgets: `label`, `button: true`, `enabled`, `value` for anything the widget
  tree cannot infer. `MergeSemantics` for a composite that should read as one thing;
  `ExcludeSemantics` for decoration.
- **Text scaling to 200% must not clip. This breaks most Flutter layouts — check it explicitly.**
  Fixed `height` on a text container is the usual cause; use constraints and let it grow.
- Test with TalkBack actually running. Reading the widget tree is not the same audit.
- Touch targets ≥48dp — `IconButton` defaults are fine, custom `GestureDetector`s usually are not.
- Focus traversal on desktop (Windows/Linux) with a real keyboard, including `Shortcuts`/`Actions`.
- Contrast checked against `ThemeData` for both light and dark schemes.

---

## Report

Group by severity, not by page. Give the WCAG success criterion number for each conformance failure
(e.g. 1.4.3 Contrast Minimum, 2.4.7 Focus Visible) — it makes the finding checkable and defensible.

Close with the single most important change, and state what you could not test.
