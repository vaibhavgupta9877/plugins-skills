---
name: landing-page-craft
description: Build or upgrade a marketing landing page, hero section, or product site to a premium, conversion-focused standard. Do NOT use for in-app product UI — that's design-system and web-design-engineering.
---

# landing-page-craft

> **Read first.** Invoke the `frontend-design` skill (plugin `frontend-design@claude-plugins-official`;
> on Linux sandboxes, `/mnt/skills/public/frontend-design/SKILL.md`) and follow it. It owns the
> aesthetic direction — palette, typefaces, the signature element. This skill owns structure,
> conversion, and page craft, and never overrides its visual choices with defaults of its own.

## Owns
- Landing and marketing page structure, hero design, section rhythm, proof, CTA placement.
- Page copy for those sections.

## Does not own
- Motion implementation → `animate/scroll-motion`, `animate/design-engineering`. This skill
  *specifies* the moment; those build it.
- In-app product UI → `web-design-engineering`.
- Token system → `design-system`. Interface copy voice rules → `ux-copy`.
- Deep perf work → `web-performance`.

---

## 1. Intake — one batch, then build

Gather these before designing. Ask what is missing in **one** batch, not one question at a time. If
the user cannot answer, make a reasonable assumption, state it in a line, and continue — do not stall.

- **The one primary action.** Trial, demo, buy, waitlist, download. Exactly one.
- **The offer.** What they get, precisely.
- **The ICP**, the problem they are trying to solve, and their **top three objections**.
- **Traffic source** — ads, search, social, email — and what they already know when they land.
- **Proof available**: real logos, testimonials, numbers, case studies. What actually exists.
- **Constraints**: brand voice, existing design direction, mobile priority.

If `docs/product-direction.md` exists, read its Brand foundation, `Design language`, and matching surface section when present. The Design language's premise, refusals, type, colour, and motion signature govern every visual choice on the page. If no surface section exists, rely on the current brief and existing site; the missing section does not block page craft. Reuse verified claims, keep assumptions provisional, and do not repeat answered intake. Invoke `experience-direction` only when the user requests a separate surface-direction brief. This skill owns marketing-page structure and conversion decisions.

---

## 2. The hero is a thesis

This is the core of the skill, and it comes straight from `frontend-design`: **open with the most
characteristic thing in the subject's world** — a headline, an image, a live demo, an interactive
moment, whatever the subject calls for.

For a premium/award-grade page, choose the hero *shape* from the premise — type-as-image, artifact,
world, roster, or maxim — using `reference/premium-section-patterns.md`, which also covers
numbered buckets, index rows, method steps, and footer-as-finale. If the page has no design language
and the user wants it distinctive, route to `product-direction/art-direction` first.

**"Big number + small label + gradient accent" is the template answer.** Use it only if it is
genuinely the best option for this brief, and be able to say why.

Above the fold must communicate **what this is, in under 5 seconds, on a phone**:

1. Headline — outcome plus audience.
2. Subheadline — one or two sentences; what it is and who it is for.
3. One primary CTA. **Never competing CTAs above the fold.**
4. One proof signal — a real logo strip, a real number, a short real testimonial.
5. Hero visual — the product itself where possible.

**Match message to source**: if traffic comes from an ad, mirror the ad's headline and promise.

---

## 3. Section architecture

Each section has a job. **Cut any section without one.**

| # | Section | Job |
|---|---|---|
| 1 | Hero | One claim, one action |
| 2 | Proof | Earn the right to be believed, immediately |
| 3 | The problem, in the user's words | Recognition — they see themselves |
| 4 | How it works, **3 steps max** | Remove "how would this actually work for me" |
| 5 | Differentiation | Why this and not the obvious alternative |
| 6 | Objection handling / FAQ | Answer the three objections from intake |
| 7 | Pricing | Remove the last unknown |
| 8 | Final CTA | Identical promise to the hero |

Order is a default, not a law. High-friction offers move the FAQ earlier. High-intent traffic
(a known-user email) can drop to hero + proof + CTA.

**Proof sits next to the claim it supports**, never pooled in a testimonial graveyard at the bottom.

Build **section by section**, in order, rather than regenerating the whole page each iteration — it
keeps diffs reviewable and control in the user's hands.

---

## 4. Copy that converts

- **Benefit first, feature second.** Features are what it does; benefits are what that means.
- **Be specific.** Not "save time and streamline" — "cut weekly reporting from 4 hours to 15 minutes".
- **CTA = verb + what they get.** "Start free trial", "Book a demo". Never "Learn more" or "Submit".
- **Reduce risk** explicitly: free plan, no card required, cancel anytime, or a guarantee. Pick one.
- Headline shapes that work: "{Outcome} without {pain}" · "The {category} for {audience}" ·
  "Ship {result} in {time}".
- Voice rules come from `ux-copy` and `frontend-design`. Copy makes a design feel as templated as the
  design itself.
- **Never fabricate.** No invented stats, no fake logo walls, no dummy testimonials, no round fake
  numbers. Where proof does not exist, use an honest placeholder that says real material is needed
  and tell the user. Fabricated credibility is the one failure a visitor can actually catch.

---

## 5. One signature moment

**Specify exactly one**, then hand implementation to `animate/scroll-motion` (if scroll-linked) or
`animate/design-engineering` (if interaction-linked).

**Every section having an effect is the failure mode.** It is also the most reliable tell that a page
was generated rather than designed. Reveals elsewhere stay subtle and fast: 200–300ms, 8–24px
translate, opacity — never 600ms fades and large slide-ins. The exception is a design language with
an editorial or cinematic tempo: headline line-mask reveals may run 600–900ms (see
`animate/scroll-motion` Step 8); body copy and cards stay fast.

Write the moment as a spec — what it is, what it communicates, where it sits — not as an
implementation.

---

## 6. Performance is a conversion feature

**A beautiful page that loads slowly converts worse than a plain page that loads fast.**

Non-negotiable baseline:
- **LCP under 2.5s** on real hardware and a throttled connection.
- **No CLS** from webfonts or images: `next/font` with `display: swap` and a metric-matched fallback;
  explicit `width`/`height` or `aspect-ratio` on every image.
- `next/image` with correct `sizes`, and `priority` on the LCP image only.
- No render-blocking third-party script above the fold. Analytics loads after.
- The hero must not depend on client-side fetching.

Hand anything deeper to `web-performance`.

---

## 7. SEO baseline

- Exactly one `h1`; heading order descends without skipping.
- Metadata via the App Router `metadata` export: title, description, canonical.
- An Open Graph image that is actually designed, not a screenshot.
- Structured data where applicable (`FAQPage` for the FAQ, `Organization`, `Product`).
- **Do not index** ad-only campaign pages or time-bound offers — `noindex` them. Do index evergreen
  offers where search intent matches the promise.
- FAQ in plain question-and-answer form, which is also what answer engines quote.

---

## 8. guptatechnologies.in specifically

**An agency or product-company site is judged as a portfolio piece.** Its own craft is the proof.
Every argument this skill makes about restraint, performance, and detail applies twice over: a
prospective client reads the site's execution as a direct sample of the work they would be buying.
A generic template here costs more than a generic template anywhere else.

---

## 9. Ship checklist

- [ ] Verified on a **real phone**, not just a responsive-mode viewport.
- [ ] Keyboard navigable; visible focus; skip link.
- [ ] No layout shift on load; fonts and images reserved.
- [ ] Every CTA actually works; no `href="#"` left anywhere.
- [ ] Forms validate inline, show errors specifically, and confirm on success.
- [ ] Analytics fires on page view and on the primary CTA.
- [ ] Metadata, OG image, favicon, and a custom 404.
- [ ] Privacy policy and terms linked in the footer.
- [ ] Alt text on every meaningful image; semantic `nav`/`main`/`section`.
- [ ] Reduced-motion respected on the signature moment.

## Sources
`frontend-design` (Anthropic); elayadesign `landing-page-design` (structure, conversion rules,
nested-radius and spacing discipline); `build-awwwards-quality-sites` (art direction and honest asset
sourcing). Aesthetic prescriptions in those sources are deliberately not carried over — direction
comes from `frontend-design` per brief.
