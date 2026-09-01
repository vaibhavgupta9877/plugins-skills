---
name: web-design-engineering
description: Build application UI screens and components in Next.js + Tailwind + shadcn — layout, composition, responsive behaviour, state rendering. Do NOT use for marketing pages (landing-page-craft) or token systems (design-system).
---

# web-design-engineering

> **Read first.** Invoke the `frontend-design` skill (plugin `frontend-design@claude-plugins-official`;
> on Linux sandboxes, `/mnt/skills/public/frontend-design/SKILL.md`) and follow it. This skill adds
> Next.js and Tailwind execution on top of its aesthetic direction and never contradicts it.

## Owns
- In-app screen composition, layout, responsive strategy, component assembly.
- Rendering every UI state.

## Does not own
- Tokens and the component system → `design-system`.
- Marketing pages → `landing-page-craft`.
- Motion and interaction feel → `animate/design-engineering`, `animate/animate`.
- Data fetching, procedures, server logic → `next-saas/*`.
- Interface wording → `ux-copy`.

---

## 1. Load the token system first

**Read `design-system`'s tokens before writing a single style.** If no system exists, build the
minimum token set — surface, text, text-muted, border, accent, danger, one type scale, one spacing
scale — and *then* build the screen. Hardcoding values "for now" is how a codebase ends up with four
greys and no way to ship dark mode.

---

## 2. Five states, or the screen is incomplete

**Every screen renders five states: loading, empty, populated, error, and partial/degraded.** Most
UI ships only the populated state, and every bug report about "it just showed nothing" traces back
to that.

- **Loading** — a skeleton shaped like the real layout, not a spinner and not generic grey bars.
  Delay it 200–500ms so a fast response never flashes it.
- **Empty** — **designed as an invitation to act**, not a blank panel. Say what goes here and give
  the action that creates the first one. First-run empty and filtered-to-nothing empty are different
  screens with different messages.
- **Populated** — including the awkward cases: one item, 10,000 items, a 60-character name, a missing
  avatar, a negative number.
- **Error** — recoverable in place, keeping the user's input, with retry next to what failed. Never a
  whole-page replacement for a failed widget.
- **Partial / degraded** — some data loaded, some failed; offline; stale-while-revalidating. Show
  what you have and mark what you don't.

Build them in this order. Building populated first and retrofitting the rest is how the retrofit
never happens.

---

## 3. Server and client components

- **Default to server.** Mark `"use client"` at the **smallest leaf that needs interactivity** — the
  button with the handler, not the page that contains it.
- **`"use client"` at the top of a page defeats the architecture.** The whole subtree ships to the
  browser, and the reason for choosing this stack is gone. Treat it as a defect in review.
- **Pass server data down as props** rather than fetching in client components. A client component
  that fetches what its server parent already had is a waterfall.
- Keep client components leaf-shaped: they can *receive* server-rendered children, so an interactive
  wrapper does not force its contents to be client too.
- Suspense boundaries around slow regions, with the skeleton as the fallback — that is streaming, and
  it is the cheapest perceived-performance win in the stack.
- `loading.tsx` and `error.tsx` per route segment, both actually designed.

---

## 4. Responsive

- **Design the narrow layout first and let it expand.** Retrofitting mobile onto a desktop layout
  produces the hidden-overflow, three-nested-scrollbar result every time.
- **Breakpoints derive from where the content breaks**, not from device names. If the card grid gets
  awkward at 720px, the breakpoint is 720px, whatever device that is.
- **Container queries over viewport queries for reusable components.** A card in a sidebar and the
  same card in a main column should respond to their container, not to the window. This is the single
  biggest improvement available to a component library.
- Fluid type and spacing (`clamp()`) between breakpoints beats four discrete jumps.
- Test at 320px, at the awkward 700–900px middle, and at 1920px. The middle is where layouts break
  and where nobody looks.

---

## 5. Composition over configuration

- **Compound components** (`Card`, `Card.Header`, `Card.Body`) over a component with fifteen props.
- Pass `children` and slots instead of adding `renderHeader`, `headerIcon`, `headerAction`.
- Lift a component out of a page only when it is used twice, or when the page file has stopped being
  readable. Premature extraction is as costly as none.
- Colocate: a component used by exactly one route lives with that route, not in a global
  `components/` bucket.
- One component, one job. A component that fetches, transforms, and renders is three.

---

## 6. Layout stability

- Reserve space for async content — `min-height` on the region, or the skeleton at the real size.
- `aspect-ratio` on every image and media embed.
- Avoid conditional rendering that changes height with no transition; a banner appearing above
  existing content pushes what the user was reading.
- `scrollbar-gutter: stable` so content doesn't shift when a scrollbar appears.
- Tabular numerals on numbers that change in place.

---

## 7. Defects to actively avoid

These recur, and each has a specific fix:

- **Nested scroll containers.** Almost always a mistake — one scroll region per screen, and the page
  itself is usually the right one.
- **`100vh` on mobile.** The URL bar makes it wrong. Use `100dvh` (or `svh`/`lvh` deliberately).
- **z-index without a scale.** Define layers as tokens (`--z-dropdown`, `--z-modal`, `--z-toast`).
  `z-50` sprinkled around is how a modal ends up under a sticky header.
- **Absolute positioning where flex or grid works.** It survives exactly one content change.
- **Text that cannot wrap.** `min-width: 0` on flex children, `overflow-wrap: anywhere` on
  user-generated strings; otherwise one long email address breaks the layout.
- Fixed heights on anything containing text — it will be translated, or scaled to 200%.
- `overflow: hidden` used to hide a layout bug rather than fix it.
- Grid templates hardcoded to today's number of columns.

---

## 8. Before you finish

- Tab through the screen. Every control reachable, focus visible, order matches the visual order.
- Look at all five states.
- Check both themes.
- Check 320px and 200% zoom.
- Confirm no `"use client"` sits higher than it needs to.

Then hand the interaction polish to `animate/design-engineering` and the review to
`critique/ui-critique`.

## Sources
`frontend-design` (Anthropic); ConardLi `web-design-engineer`; MengTo `web-design`; jakubkrehel
`better-ui`; `tastemaker`.
