# Plugin: `web-ui`

**Enable for:** Next.js SaaS projects and guptatechnologies.in.
**Purpose:** the visual and structural layer of web UI. Not motion (`animate`), not application architecture (`next-saas`).
**Skills:** 5

**Every skill here opens with the R4 `frontend-design` directive.** That file is the aesthetic authority; these skills add Tailwind/shadcn/Next-specific execution on top of it.

**Stack assumption:** Next.js App Router + TypeScript + Tailwind + shadcn/ui + Motion. Detect and follow the repo's actual setup; do not impose this if the repo differs.

When `docs/product-direction.md` exists, relevant web skills consume its approved brand/surface context conditionally. `product-direction` sets direction; `web-ui` owns web implementation and execution details.

---

## `design-system` — CORE

**description:** `Define, extend, or audit the design token system and component library for a web project — colour, type scale, spacing, radii, elevation, component variants. Do NOT use for one-off page layout — that's web-design-engineering.`

**Owns:** tokens, scales, theming, component API design, variant systems, dark mode, documentation of the system.
**Does NOT own:** aesthetic direction → `/mnt/skills/public/frontend-design/SKILL.md`; page composition → `web-design-engineering`; visual review → `critique/ui-critique`.

**Body must cover:**
- **[R4] frontend-design directive** — the palette and type *choices* come from there. This skill turns those choices into a system.
- If `docs/product-direction.md` exists, read approved Brand foundation values; voice/category inform role names and density, not palette/type/radius/elevation. Defer unresolved visual values to `frontend-design` or ask.
- Layered tokens, three levels: primitive (`blue-500`, `space-4`) → semantic (`color-surface`, `color-danger`, `space-section`) → component (`button-padding-x`). Components consume semantic tokens only. A component referencing a primitive token is a defect.
- Tailwind v4: define tokens as CSS custom properties in `@theme`. Never hardcode a hex or a px value in a component. Grep for hardcoded values as the primary audit action.
- Type scale: pick a ratio and commit. Define role names (display, heading-1..3, body, caption, mono/data) rather than sizes at call sites.
- Spacing: one scale, applied consistently. Inconsistent spacing is the most visible symptom of a missing system.
- shadcn/ui: it is a starting point that must be *customised*, not a finished look. Unmodified shadcn components are an AI-generated tell. Restyle the primitives to match the token system before shipping.
- Component API rules: variants over booleans (`variant="danger"` not `isDanger`), `cva` for variant definition, forward refs, accept `className`, sensible defaults, no required props with obvious defaults.
- Dark mode: designed, not inverted. Semantic tokens make this cheap; check contrast in both.
- Audit mode: when asked to audit rather than build, output the hardcoded-value list, the naming inconsistencies, and the components that duplicate an existing one.

**Absorb:** `/mnt/skills/user/design-system`, `/mnt/skills/plugins/design:design-system`, token/system portions of `/mnt/skills/user/t3-frontend-craft`.

**reference/:** `tailwind-tokens.md` — the `@theme` template and cva patterns.

**Budget:** 200 lines.

---

## `landing-page-craft` — CORE

**description:** `Build or upgrade a marketing landing page, hero section, or product site to a premium, conversion-focused standard. Do NOT use for in-app product UI — that's design-system and web-design-engineering.`

**Owns:** landing/marketing page structure, hero design, section rhythm, social proof, CTA placement, conversion structure.
**Does NOT own:** motion implementation → `animate/*` (this skill *specifies* the motion moment; `animate` and `scroll-motion` build it); app UI → `web-design-engineering`; marketing copy strategy → this skill owns page copy, but voice rules come from `ux-copy`.

**Body must cover:**
- **[R4] frontend-design directive.** The hero-as-thesis principle is the core of this skill — restate that the hero must open with the most characteristic thing about the product, and that "big number + small label + gradient accent" is the template answer.
- Section architecture that works, with the reasoning for each: hero (one claim, one CTA) → proof (concrete, not logos-you-don't-have) → the problem stated in the user's words → how it works (3 steps max) → differentiation → objection handling → pricing → final CTA. Cut any section without a job.
- **One** signature moment per page. Specify it, then hand implementation to `animate/scroll-motion` or `animate/design-engineering`. Every section having an effect is the failure mode.
- Above-the-fold must communicate what the product is in under 5 seconds, on a phone.
- Performance is a conversion feature: LCP under 2.5s, no CLS from webfonts or images, images via `next/image` with correct sizes, fonts via `next/font` with `display: swap`. A beautiful page that loads slowly converts worse than a plain one that loads fast. Hand deeper work to `web-performance`.
- SEO baseline: one `h1`, semantic heading order, metadata via the App Router metadata API, Open Graph image, structured data where applicable.
- **guptatechnologies.in context:** an agency/product-company site is judged as a portfolio piece. Its own craft is the proof. State this.
- Ship checklist: mobile-first verified on a real device, keyboard navigable, no layout shift, CTA works, forms validate and show errors, analytics fires.

**Absorb:** `/mnt/skills/user/str-landing-setup`.
**Sources:** `github.com/elayadesign/ai-design-skills/tree/main/skills/landing-page-design`, `build-awwwards-quality-sites` (fetch by name).

**Budget:** 200 lines.

---

## `web-design-engineering` — CORE

**description:** `Build application UI screens and components in Next.js + Tailwind + shadcn — layout, composition, responsive behaviour, state rendering. Do NOT use for marketing pages (landing-page-craft) or token systems (design-system).`

**Owns:** in-app screen composition, layout, responsive strategy, component assembly, rendering every UI state.
**Does NOT own:** tokens → `design-system`; motion → `animate/*`; data fetching and server logic → `next-saas/*`.

**Body must cover:**
- **[R4] frontend-design directive.**
- Load `design-system` tokens before writing any styles. If no system exists, build the minimum token set first rather than hardcoding.
- **Every screen renders five states or it is incomplete:** loading, empty, populated, error, and partial/degraded. Design the empty state as an invitation to act. Most UI ships only the populated state.
- Responsive: design the narrow layout first and let it expand. Breakpoints derive from where the content breaks, not from device names. Container queries over viewport queries for reusable components.
- Composition over configuration: build compound components (`Card`, `Card.Header`) rather than components with fifteen props.
- Server vs client components: default to server, mark `"use client"` at the smallest possible leaf. A `"use client"` at the top of a page defeats the architecture. Pass server data down as props rather than fetching in client components.
- Layout stability: reserve space for async content, use `aspect-ratio` for media, avoid conditional rendering that changes height without transition.
- Common defects to actively avoid: nested scroll containers, `100vh` on mobile (use `100dvh`), z-index without a scale, absolute positioning where flex/grid works, text that can't wrap.

**Absorb:** `/mnt/skills/user/t3-frontend-craft` (UI/composition portions; motion portions → `animate/design-engineering`; setup portions → `next-saas/t3-architecture`).
**Sources:** `github.com/ConardLi/garden-skills/tree/main/skills/web-design-engineer`, `github.com/MengTo/Skills/tree/main/agent-skills/web-design`, `github.com/jakubkrehel/skills` (`better-ui`), `github.com/codeswithroh/tastemaker`.

**Budget:** 200 lines.

---

## `ux-copy` — SUPPORTING

**description:** `Write interface copy — button labels, error messages, empty states, onboarding, confirmations, form help text. Do NOT use to review existing copy, that's critique/copy-critique.`

**Owns:** authoring interface text.
**Does NOT own:** reviewing → `critique/copy-critique`; marketing/landing headlines → `landing-page-craft`.

**Body must cover:**
- **[R4] Read `/mnt/skills/public/frontend-design/SKILL.md`, "More on writing in design."** It is the standard; this skill supplies patterns and examples, not competing rules.
- Pattern library with before/after pairs: buttons and CTAs · confirmation dialogs (the button says the action, never "OK") · error messages (what happened + how to fix, no apology, never vague) · empty states · loading text · success toasts (past tense of the action's own verb) · destructive confirmations · form validation · permission requests.
- Consistency rule: an action keeps one name from control to toast to history entry.
- Name things by what the user controls, never by system internals.
- Write the copy in the product's actual voice — detect it from existing screens before inventing one.

**Absorb:** `/mnt/skills/plugins/design:ux-copy`.

**Budget:** 100 lines.

---

## `web-performance` — SUPPORTING

**description:** `Diagnose and fix web performance — Core Web Vitals, bundle size, render blocking, image and font loading in Next.js. Use when a page is slow or before a launch.`

**Owns:** LCP/CLS/INP diagnosis, bundle analysis, asset strategy, render-blocking removal.
**Does NOT own:** animation jank → `animate/improve-animations`; server/DB performance → `next-saas/deploy-observability`.

**Body must cover:**
- Measure before optimising, on real hardware and throttled network. Lighthouse on a desktop dev machine is misleading. Prefer field data (`web-vitals`, Vercel Analytics) over lab scores.
- LCP: identify the actual LCP element first — it is usually a hero image or a webfont-rendered heading. Fixes: `priority` on the LCP image, correct `sizes`, preconnect, `next/font` self-hosting, no client-side fetch for above-fold content.
- CLS: reserve dimensions on all media, `font-display: swap` with a matched fallback metric, no content injected above existing content.
- INP: the modern killer. Break up long tasks, defer non-critical JS, avoid heavy client components, debounce expensive handlers.
- Bundle: run `@next/bundle-analyzer`. Usual offenders are date libraries, icon barrels imported wholesale, chart libraries, and a `"use client"` boundary set too high.
- Server components and streaming (`loading.tsx`, `Suspense`) are the highest-leverage Next.js perf tool — use them before micro-optimising.
- State the budget: LCP < 2.5s, CLS < 0.1, INP < 200ms, initial JS < 200KB gzipped. Report pass/fail against these.

**Budget:** 120 lines.
