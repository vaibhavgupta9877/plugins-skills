---
name: design-engineering
description: Build the interaction detail that makes an interface feel hand-crafted — hover, press, focus, drag, optimistic and loading states. Do NOT use for layout or visual design — that's web-ui/*.
---

# design-engineering

> **Read first.** Invoke the `frontend-design` skill (plugin `frontend-design@claude-plugins-official`;
> on Linux sandboxes, `/mnt/skills/public/frontend-design/SKILL.md`) and follow it. This skill is the
> craft layer underneath its aesthetic direction, never a replacement for it.

## Owns
- Interaction states, perceived performance, and the micro-detail between design and engineering.

## Does not own
- Visual and layout design → `web-ui/web-design-engineering`, `web-ui/design-system`.
- Pure animation timing decisions → `animate`.
- Scroll-linked motion → `scroll-motion`.
- Judging the result → `critique/ui-critique`, `critique/motion-critique`.

**This is the highest-leverage skill in the plugin for "doesn't look AI-generated."** Unseen details
compound: no single one is noticed, and their absence is felt as cheapness.

---

## 1. Five states, not two

**Every interactive element has five states, and most implementations ship two.**

`default` · `hover` · **`active` / press** · **`focus-visible`** · `disabled`

Plus `loading` and `error` for anything async.

**The missing press state is the most common reason a UI feels cheap.** It is one line:

```css
.button { transition: transform 160ms var(--ease-out); }
.button:active { transform: scale(0.97); }   /* 0.95–0.98; subtle */
```

Apply it to every pressable thing — buttons, cards, list rows, icon buttons, tabs. Instant feedback
makes the interface feel like it is listening.

Rules for the rest:
- `:focus-visible`, not `:focus` — no ring on mouse click, always a ring on keyboard. `outline: none`
  with no replacement is an accessibility failure, not a style choice.
- The focus ring is designed: a token colour, an offset, and 3:1 contrast against both the element
  and the background.
- Hover motion gated with `@media (hover: hover) and (pointer: fine)` — touch fires false hovers.
- `disabled` must still be readable, and must say *why* it is disabled somewhere reachable. A
  disabled button with no explanation is a dead end.

---

## 2. Perceived performance beats measured performance

The user experiences latency, not milliseconds.

- **Instant feedback on press, before the network resolves.** The press state fires on pointer-down,
  not on the response.
- **Optimistic updates** for any mutation that almost always succeeds: apply locally, reconcile on
  response, roll back visibly with an explanation on failure. Never roll back silently.
- **Skeletons shaped like the real layout**, not generic grey bars and not a spinner. A skeleton
  starts answering "what am I waiting for" immediately; a spinner says only "wait". Match the real
  content's line lengths and card shapes, or it produces layout shift when the data lands.
- **Delay the loading indicator 200–500ms**, so a fast response never flashes one.
- Prefetch on hover or focus for navigation the user is clearly about to take.
- Keep the previous data visible while refetching, dimmed or with a subtle busy marker, instead of
  collapsing to a skeleton on every refresh.

---

## 3. Async state discipline

**Four states, all designed: `idle` → `pending` → `success` → `error`.** Most code designs one and a
half.

- `pending`: the triggering control shows it, and stays in place. Never let the button move or resize
  — swap the label for a spinner at the same width, or overlay it.
- `success`: confirm in the past tense of the action's own verb. "Publish" produces "Published".
- **`error`: recoverable in place, never a page replacement.** Keep the user's input. Say what broke
  and how to fix it. Put the retry next to the thing that failed.
- Disable double submission, and make it idempotent underneath rather than relying on the disable.
- Empty is a fifth state and belongs to the screen, not the control — see
  `web-ui/web-design-engineering`.

---

## 4. Detail that signals craft

- **Cursor matches the affordance**: `pointer` on anything clickable, `text` on text, `grab`/`grabbing`
  on draggables, `not-allowed` on disabled, `col-resize` on a resizer.
- **Disable text selection on controls only** (`user-select: none` on buttons, tabs, labels) — never
  on content, where it breaks copying.
- **Hit areas larger than visual bounds.** A 16px icon button gets a 44px hit area via padding or a
  pseudo-element, without changing the layout.
- **Tabular numerals** (`font-variant-numeric: tabular-nums`) on anything that changes in place —
  counters, timers, prices, tables. Otherwise the digits jitter as widths change.
- **No layout shift when content loads.** Reserve space: `aspect-ratio` on media, `min-height` on
  async regions, a fixed-width container for changing text.
- **Scrollbar-stable gutters** (`scrollbar-gutter: stable`) so content doesn't jump when a scrollbar
  appears.
- Keyboard shortcuts on frequent actions, discoverable in the tooltip or menu that already exists.
- Tooltips: delay 300–700ms on first hover, then **instant with no animation for adjacent tooltips**
  while one is already open. It makes the whole toolbar feel faster.
- Autofocus the first meaningful field in a dialog, and never a field the user must scroll to reach.
- Enter submits a single-input form; Cmd/Ctrl+Enter submits a textarea form. Escape closes.
- Trim and normalise pasted input rather than rejecting it — a pasted card number with spaces should
  just work.

---

## 5. Drag and gesture

- **Follow the pointer 1:1** while dragging. Any smoothing during the drag reads as lag.
- **Resist at boundaries** — rubber-band with diminishing returns, never a hard stop.
- **Spring on release**, seeded with the release velocity. See `animate/reference/spring-presets.md`.
- **Dismiss on velocity, not only distance**: a fast short flick dismisses; a slow long drag that
  stops short snaps back.
- **Cancel cleanly** on Escape and on pointer-cancel, returning with the same spring.
- Set `touch-action` so the browser knows which axis you own; without it, a horizontal drag fights
  vertical scrolling on mobile.
- Show the drop target before the drop, not after.

---

## 6. Restraint

Every item above is cheap and invisible. That is the point — they compound into "this feels
expensive" without any single one being noticed.

None of it substitutes for the one deliberate moment the design is remembered by. If you find
yourself adding a sixth interactive flourish to a screen, you are decorating, not engineering.
Remove one accessory before you finish.

## Sources
`frontend-design` (Anthropic); Emil Kowalski's design-engineering writing; jakubkrehel `better-ui`;
`tastemaker`; ConardLi `web-design-engineer`; MengTo `web-design`.
