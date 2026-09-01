---
name: scroll-motion
description: Build scroll-triggered reveals, scroll-scrubbed sequences, parallax, and cinematic scroll storytelling on the web. Do NOT use for non-scroll animation — that's animate.
---

# scroll-motion

> **Read first.** Invoke the `frontend-design` skill (plugin `frontend-design@claude-plugins-official`;
> on Linux sandboxes, `/mnt/skills/public/frontend-design/SKILL.md`) and follow it — particularly
> "spend your boldness in one place". This skill applies that rule to scroll.

## Owns
- Everything scroll-linked on the web: reveals, scrubbed sequences, pinning, parallax, narrative.

## Does not own
- General motion → `animate`.
- Flutter scroll effects → `flutter-motion-craft`.
- Judging existing scroll motion → `critique/motion-critique`.
- Page structure and section order → `web-ui/landing-page-craft` (it specifies the moment; this skill
  builds it).

---

## Step 1 — Choose the mode deliberately

Two modes. Picking the wrong one is the root cause of most bad scroll work.

**Triggered** — fires once when an element enters the viewport, then it is done. `IntersectionObserver`.
Cheap, robust, degrades gracefully. **Correct for ~90% of cases**, including every reveal.

**Scrubbed** — progress bound to scroll position, reversing when the user scrolls back. Expensive,
powerful, and only justified for a deliberate narrative moment.

State which you chose and why, in one line, before writing code. If the answer is "triggered", most
of the rest of this skill does not apply — build it and stop.

---

## Step 2 — Non-negotiables

- **Never hijack scroll.** No `preventDefault` on wheel, no scroll-jacking to snap between sections,
  no disabling native scrolling. The user's scroll belongs to the user.
- **Momentum, trackpad, wheel, touch, and keyboard must all behave. Test all four inputs** — a
  trackpad delivers many small deltas, a wheel delivers few large ones, and touch delivers momentum
  after the finger lifts. Code tuned on one feels broken on the others.
- **Content must be fully readable with JS disabled and under `prefers-reduced-motion`.** Content
  that only exists after a scroll trigger is a content bug, not a motion decision. Start from visible
  and let JS add the motion, or ship the reveal via `@starting-style` / CSS only.
- Never read layout inside a scroll handler.

---

## Step 3 — Reveals (the common case)

**Subtle and fast.** 200–300ms, an 8–24px translate, plus opacity. Trigger once, then unobserve.

```js
const io = new IntersectionObserver((entries) => {
  for (const e of entries) {
    if (!e.isIntersecting) continue;
    e.target.dataset.revealed = "true";
    io.unobserve(e.target);          // once. Re-revealing on scroll-back is nausea, not delight
  }
}, { rootMargin: "0px 0px -10% 0px", threshold: 0.1 });
```

```css
[data-reveal] { opacity: 0; transform: translateY(16px);
                transition: opacity 250ms var(--ease-out), transform 250ms var(--ease-out); }
[data-reveal][data-revealed] { opacity: 1; transform: none; }
@media (prefers-reduced-motion: reduce) {
  [data-reveal] { opacity: 1; transform: none; transition: none; }
}
```

**Large slide-ins, 600ms-plus fades, and blur-heavy reveals are the standard AI tell.** So is
revealing every section on a page. Reveal what benefits from arriving; let the rest simply be there.

Stagger a group at 30–80ms per item, capped around 300ms total, and never on more than ~12 items.

---

## Step 4 — Scrubbed sequences

Budget **one per page**. This is "spend your boldness in one place" applied to scroll.

The shape, every time:

1. **Pin the section** for a defined scroll distance (`position: sticky` on a taller parent is the
   cheap, robust way; a library's pin is fine if one is already in the project).
2. **Map scroll position to a normalised 0–1 progress** for that section.
3. **Drive `transform` and `opacity` from that progress.** Nothing else.

```js
const track = document.querySelector("[data-track]");   // tall parent
const stage = document.querySelector("[data-stage]");   // sticky child
let ticking = false;

function onScroll() {
  if (ticking) return;
  ticking = true;
  requestAnimationFrame(() => {
    const r = track.getBoundingClientRect();
    const total = r.height - window.innerHeight;
    const p = Math.min(1, Math.max(0, -r.top / total));   // 0..1
    stage.style.setProperty("--p", p.toFixed(4));
    ticking = false;
  });
}
addEventListener("scroll", onScroll, { passive: true });
```

Then read `--p` in CSS. Do the geometry read once per frame inside the rAF, never per element.

Where `animation-timeline: view()` / `scroll()` is available and the sequence is simple, prefer it —
it runs off the main thread. Feature-detect and fall back to the handler above.

Give the pinned section a real reason to exist: a product explained step by step, a process shown in
sequence. A pinned section that only moves things around costs the user scroll distance and returns
nothing.

---

## Step 5 — Parallax

Layer offsets driven from the same normalised progress. Keep the differential small — a few percent
of viewport height. Large parallax reads as a template, and on a long page it drifts far enough to
break the layout.

Never parallax text the user is reading. Backgrounds and decorative layers only.

---

## Step 6 — Performance

- `IntersectionObserver` over scroll listeners, always, where the need is "did it enter view".
- An unavoidable scroll listener is `{ passive: true }` plus rAF throttling, with **one** layout read
  per frame.
- Compositor-only properties. A scrubbed sequence animating `top` or `height` will jank on every
  device that isn't yours.
- Long pages: unobserve after reveal, and disconnect observers on unmount. Hundreds of live observers
  is its own cost.
- Images inside a pinned sequence must be sized and preloaded — a scrubbed sequence that loads while
  scrubbing looks broken.

---

## Step 7 — Mobile

**Scrubbed scroll behaves differently under iOS momentum**: scroll events are throttled or suppressed
during the momentum phase, so a scrub can freeze then jump.

Decide explicitly, and say which you chose:
- **Simplify to triggered reveals below a breakpoint** — the safe default, and usually the right one.
- Or keep the scrub and **test on a real device**, not a desktop responsive-mode.

Also check: `100vh` on mobile is wrong once the URL bar collapses — use `100dvh` for a pinned stage,
or the pin will shift mid-scroll.

---

## Output

Say which mode you used and why, name the one scrubbed moment if there is one, and confirm the three
degradations: JS disabled, reduced motion, and small viewport.

## Sources
`frontend-design` (Anthropic); scroll-storytelling and animation-on-scroll patterns as concept
sources, distilled rather than vendored.
