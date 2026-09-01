---
name: web-performance
description: Diagnose and fix web performance — Core Web Vitals, bundle size, render blocking, image and font loading in Next.js. Do NOT use for server or deploy-side performance — that's next-saas/deploy-observability.
---

# web-performance

## Owns
- LCP, CLS, and INP diagnosis and repair.
- Bundle analysis, asset strategy, render-blocking removal.

## Does not own
- Animation jank and dropped frames → `animate/improve-animations`.
- Server, database, and deploy performance → `next-saas/deploy-observability`,
  `next-saas/trpc-prisma-patterns` (N+1 queries).
- Release gating → `secure-ship/prod-readiness-gate`.

---

## Step 1 — Measure first, on real conditions

**Do not optimise before measuring.** The bottleneck is rarely where it feels like it is.

- **Real hardware, throttled network.** A Lighthouse run on a desktop dev machine over localhost is
  misleading to the point of being harmful — it will tell you a page is fast that takes eight seconds
  on a mid-range Android over 4G.
- **Prefer field data over lab scores.** Vercel Analytics or the `web-vitals` library reports what
  users actually experience, across their devices; Lighthouse reports one synthetic run.
- Test the **slowest real page**, not the homepage.
- Record the numbers before you change anything, so you can prove the change worked.

**Budgets — report pass/fail against these:**

| Metric | Budget |
|---|---|
| LCP | < 2.5s |
| CLS | < 0.1 |
| INP | < 200ms |
| Initial JS | < 200KB gzipped |

---

## Step 2 — LCP

**Identify the actual LCP element first.** Chrome DevTools Performance panel names it. It is usually
a hero image or a heading rendered in a webfont — guessing wastes the whole session.

Then, in order of typical payoff:

- `priority` on the LCP image (and **only** that one — `priority` on everything is the same as on
  nothing). Correct `sizes` so the browser doesn't download a 2400px image for a 400px slot.
- Serve modern formats; let `next/image` handle AVIF/WebP negotiation.
- **`next/font` to self-host**, which removes a DNS lookup, a connection, and a redirect to
  fonts.gstatic.com from the critical path. Subset it.
- `preconnect` to any origin the LCP resource genuinely comes from.
- **No client-side fetch for above-the-fold content.** A hero that renders after a `useEffect` fetch
  cannot have a good LCP by construction. Server component or static.
- Remove render-blocking CSS and synchronous third-party scripts above the fold.
- Check the TTFB portion separately — if the server takes 1.5s, no front-end fix will save the page.

---

## Step 3 — CLS

- **Dimensions on all media.** `width`/`height` or `aspect-ratio` on every image, iframe, video, and
  ad slot.
- **Fonts**: `display: swap` plus a fallback whose metrics are adjusted to match
  (`size-adjust`/`ascent-override`, which `next/font` generates automatically). An unmatched fallback
  is the single most common CLS source on a well-built site.
- Never inject content above existing content — banners, cookie notices, and validation summaries
  push what the user is reading. Reserve the space or overlay it.
- Reserve space for async regions with `min-height` or a correctly-sized skeleton.
- `scrollbar-gutter: stable`.
- Animate only `transform` and `opacity` — animating layout properties generates CLS as well as jank.

---

## Step 4 — INP (the modern killer)

INP replaced FID and is much harder to pass, because it measures every interaction, not the first.

- **Break up long tasks.** Anything over 50ms blocks input. Yield with `scheduler.yield()` where
  available, or `setTimeout(0)`, between chunks of work.
- **Defer non-critical JS.** `next/script` with `strategy="lazyOnload"` for analytics, chat widgets,
  and anything not needed for the first interaction.
- **Avoid heavy client components.** Every KB of client JS is parse time on a slow phone. Push work
  to the server.
- Debounce expensive handlers (search-as-you-type, resize, scroll) and keep the work off the input
  path.
- Watch for re-render storms: a context provider high in the tree re-rendering the whole page on
  every keystroke is a classic INP failure. Memoise the *provider value*, not just the consumers.
- Large lists need virtualisation past a few hundred rows.

---

## Step 5 — Bundle

Run `@next/bundle-analyzer` and read it before guessing. The usual offenders, in order:

1. **A `"use client"` boundary set too high** — the single biggest cause of an oversized bundle in
   App Router apps. Push it to the leaf.
2. **Icon barrels imported wholesale** — `import { X } from "lucide-react"` is fine with modern
   tree-shaking, but `import * as Icons` is not. Verify in the analyzer rather than assuming.
3. **Date libraries** — moment and full-locale date-fns builds. Prefer `Intl.DateTimeFormat`, which
   is free and already in the browser.
4. **Chart libraries** — often larger than the rest of the app. Dynamic-import them below the fold.
5. Duplicate versions of the same package pulled by different dependencies.

`next/dynamic` for anything below the fold or behind an interaction: modals, editors, charts, maps.

---

## Step 6 — Use the framework before micro-optimising

**Server components and streaming are the highest-leverage tools in this stack, and they are free.**

- Move data fetching to server components; the data never becomes client JS.
- `loading.tsx` and `<Suspense>` around slow regions so the shell paints immediately. A page that
  streams feels fast even when the total time is unchanged.
- Static or ISR for anything not per-user.
- `fetch` caching and `revalidate` set deliberately, not left to chance.

Do this before hand-optimising a component. Reordering `useMemo` calls in a client component that
should have been a server component is wasted effort.

---

## Step 7 — Report

State the before and after numbers for each metric, against the budget, and say what conditions you
measured under. A "40% faster" with no baseline and no device is not a result.

Then name the single highest-leverage remaining item, and whether the page passes or fails its
budgets. If it still fails, say so plainly.
