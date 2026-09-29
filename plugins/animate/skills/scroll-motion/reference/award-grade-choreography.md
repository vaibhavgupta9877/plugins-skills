# Award-grade choreography (marketing narrative surfaces only)

Load this only when the page is a marketing, editorial, or portfolio narrative **and** a `Design language` section (from `product-direction/art-direction`) or the brief calls for editorial/cinematic tempo. Product UI never uses these patterns. Every pattern keeps the Step 2 non-negotiables: native scroll, content readable without JS and under reduced motion.

## 0. One ticker, one engine

Everything that moves per frame — smooth scroll, ScrollTrigger, canvas, cursor — runs on **one** rAF loop. Two loops (Lenis's own rAF plus GSAP's ticker plus a Three.js `setAnimationLoop`) is the most common source of award-site jank.

```js
import Lenis from "lenis";
import { gsap } from "gsap";
import { ScrollTrigger } from "gsap/ScrollTrigger";
gsap.registerPlugin(ScrollTrigger);

const reduce = matchMedia("(prefers-reduced-motion: reduce)").matches;
export const lenis = reduce ? null : new Lenis({ lerp: 0.1, smoothWheel: true }); // touch stays native
if (lenis) {
  lenis.on("scroll", ScrollTrigger.update);
  gsap.ticker.add((t) => lenis.raf(t * 1000));
  gsap.ticker.lagSmoothing(0);
}
// canvas + cursor subscribe here too: gsap.ticker.add(renderFrame)
```

Smooth scroll is a stance, not a default: Lenis smooths wheel input only and never replaces native scrolling. Anchor links go through `lenis.scrollTo`, and `html.lenis-stopped` must be cleared when modals close.

## 1. Line-mask headline reveal

The signature text move of 2026. Text exists in HTML; JS only adds the motion.

```js
import { SplitText } from "gsap/SplitText"; // free since GSAP 3.13
gsap.registerPlugin(SplitText);

document.querySelectorAll("[data-lines]").forEach((el) => {
  if (reduce) return;
  SplitText.create(el, {
    type: "lines", mask: "lines", autoSplit: true,  // re-splits on resize/font load
    onSplit: (self) => gsap.from(self.lines, {
      yPercent: 100, duration: 0.9, stagger: 0.07, ease: "expo.out",
      scrollTrigger: { trigger: el, start: "top 85%", once: true },
    }),
  });
});
```

- Headlines and one pull-quote only. Body paragraphs just appear (or fade 250ms).
- Wait for fonts (`document.fonts.ready`) or use `autoSplit`, or lines split at fallback metrics.
- Keep `aria-label`/unsplit text available — SplitText adds `aria` handling; verify with a screen reader.

## 2. Loading curtain with meaning

Only when the curtain says something native to the premise (a signature, notation, a count that is real). Rules:
- ≤ 1.5s, cancellable by scroll/click, **skipped on repeat visits** (`sessionStorage`) and under reduced motion.
- The hero's LCP element renders beneath it; the curtain lifts via `clip-path: inset(0 0 100% 0)` or `transform`, never by delaying HTML.
- Fail open: a timeout removes the gate even if the asset it waits on never loads (Cloudstudio fails open at 6.5s).

## 3. Page transitions

Prefer the platform first:

```css
@view-transition { navigation: auto; }                 /* cross-document, MPA */
.project-thumb { view-transition-name: var(--vt, none); }
::view-transition-group(*) { animation-duration: .7s;
  animation-timing-function: cubic-bezier(.16,1,.3,1); }
@media (prefers-reduced-motion: reduce) {
  ::view-transition-group(*), ::view-transition-old(*), ::view-transition-new(*) { animation: none; }
}
```

- Set a unique `view-transition-name` on the clicked thumbnail and the case-study hero for the shared-element morph. Names must be unique per page.
- In SPAs (Next.js App Router), wrap navigation in `document.startViewTransition` where supported; fall back to an instant route change, not a JS curtain.
- Transition ≤ 800ms. Visitors navigate portfolios fast; a 2s wipe is punished on the second click.

## 4. Fixed canvas + DOM chapters

```css
#stage { position: fixed; inset: 0; z-index: 0; }
#story { position: relative; z-index: 1; pointer-events: none; }
#story a, #story button { pointer-events: auto; }
```

- Map a single normalised progress (0–1 across the story) to camera/scene uniforms inside the shared ticker.
- Lazy-load the scene after first paint; show a designed poster frame until ready and forever on low-power devices (`navigator.hardwareConcurrency <= 4`, `saveData`, or failed WebGL/WebGPU context).
- Cap DPR at 1.5–2; pause rendering when the tab is hidden or the canvas is fully covered.
- Text over the scene gets a scrim or text-shadow for contrast on small screens (Oxigen does this under 820px).
- Reduced motion: render one composed frame, stop the loop.

## 5. Hover-revealed media on index rows

- Image follows the cursor with lerp 0.1–0.15 in the shared ticker, `transform` only, `pointer-events: none`.
- Only on `(hover: hover) and (pointer: fine)`. Touch shows a static thumbnail in the row.
- Preload row images on idle; a hover that shows a blank frame reads as broken.

## 6. Pinned narrative with ScrollTrigger

When the project already uses GSAP, `ScrollTrigger` pinning replaces the hand-rolled sticky pattern in Step 4 — same rules: one per page, progress drives `transform`/`opacity`/uniforms, `scrub: 0.6–1` for weighted feel, `anticipatePin: 1`, `invalidateOnRefresh: true`, and a `gsap.matchMedia()` branch that turns it into triggered reveals below the mobile breakpoint.

```js
gsap.matchMedia().add(
  "(min-width: 900px) and (prefers-reduced-motion: no-preference)", () => {
    gsap.timeline({ scrollTrigger: { trigger: "[data-track]", pin: "[data-stage]",
      start: "top top", end: "+=200%", scrub: 0.8, invalidateOnRefresh: true } })
      .to("[data-step]", { yPercent: -100, stagger: 1, ease: "none" });
  });   // gsap.matchMedia reverts everything automatically when the query stops matching
```

## 7. Tempo discipline

Use the tempo class declared in the design language (snappy / editorial / cinematic). Do not mix: an editorial headline reveal next to snappy 150ms card pops next to a 2s cinematic wipe reads as three sites stitched together.

## Verification

- Test wheel, trackpad, touch, keyboard (Space/PageDown/Home/End), and anchor links with Lenis on.
- Throttle CPU 4× in DevTools; the signature moment must hold ~60fps or simplify.
- Reduced motion and JS-disabled passes, plus a real mid-range Android phone for anything with a canvas.
