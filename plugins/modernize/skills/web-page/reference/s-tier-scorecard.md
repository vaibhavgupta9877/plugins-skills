# S-tier web scorecard

Score every dimension **0–3** before and after. Each score cites evidence (screenshot, `file:line`,
measured number). A dimension that does not apply is `N/A — reason`, never silently omitted.

| Score | Meaning |
|---|---|
| 0 | Missing or broken |
| 1 | Template default — works, but would have come out the same for any brief |
| 2 | Solid — deliberate, consistent, no real defects |
| 3 | S-tier — distinctive, traced to the premise, nothing left to fix at this tier |

**S-tier verdict:** every **Gate** row passes · no dimension below 2 · Hierarchy and Design language
at 3 for marketing surfaces (at least 2 for in-app screens, where restraint *is* the S-tier answer).

---

## Gates (pass/fail — a single fail blocks the verdict)

| Gate | Pass condition | Checked by |
|---|---|---|
| G1 Accessibility | Zero WCAG 2.1 AA failures; keyboard path complete; focus visible; reduced-motion honoured | `critique/a11y-review` |
| G2 Performance | Mobile LCP < 2.5s, CLS < 0.1, INP < 200ms (field p75 when available, else throttled lab — say which); initial JS < 200KB gz | `web-ui/web-performance` |
| G3 Honest proof | No fabricated stat, logo, testimonial, or customer; placeholders flagged | `web-ui/landing-page-craft` §4 |
| G4 Preservation | Every Phase 0 preservation item confirmed intact | this skill |
| G5 No regressions | Every Phase 1 "what is working" item still works; no new `blocking` findings | re-run lenses |

---

## Dimensions

| # | Dimension | Score-3 bar | Common score-1 tell |
|---|---|---|---|
| D1 | **Hierarchy / 5-second read** | On a phone: what it is, for whom, next action — in < 5s; eye lands on the primary thing first and you can say why | Everything the same weight; two equal CTAs |
| D2 | **Design language** | One premise; refusals honoured; anti-slop tells 0–2 | 3+ tells; trend collage |
| D3 | **Typography** | Named roles on one ratio; display face with character; fluid display sizes; tracking tightens with size; measure 45–75ch; `text-wrap` balance/pretty | Inter/system at 600 for everything; raw px sizes |
| D4 | **Colour and theming** | Ground/ink/accent with rationed accent; semantic tokens; both themes *designed*; contrast passes in both | Framework greys; dark mode = inversion |
| D5 | **Layout and rhythm** | Real grid with one intentional break; one spacing scale; airy/dense sections alternate; nested radii correct | Identical padding everywhere; near-miss alignment |
| D6 | **System integrity** | Zero hardcoded hex/px/inline styles in touched files; components consume semantic tokens; primitives restyled | Stock shadcn skin; `text-[19px]` |
| D7 | **States** | Interactive: default/hover/press/focus-visible/disabled (+loading/error). Screen: loading/empty/populated/error/partial, all designed | Only populated state; no press feedback |
| D8 | **Copy** | Headings say the thing; CTAs = verb + outcome; one term per concept; errors say what + fix | "Features", "Learn more", "Supercharge" |
| D9 | **Motion** | Every motion has a named purpose; one signature moment; ease-out entrances, faster exits; interruptible; reduced-motion and no-JS states designed | Every section fades up 600ms |
| D10 | **Responsive and input** | Works 320→1920 incl. the 700–900px middle; container queries on reusable components; touch/pointer/keyboard each designed; `dvh` not `vh` | Desktop layout squeezed; hover-only affordances |
| D11 | **Resilience** | Content readable without JS (marketing); slow-3G still usable; images/fonts reserved (no shift); errors recover in place | Blank until hydration |
| D12 | **Modern platform** | Uses current platform features where they remove JS or hacks, each with a fallback (`modern-web-platform.md`) | JS for what CSS now does; `z-index: 9999` |
| D13 | **SEO and sharing** (marketing) | One `h1`; ordered headings; metadata + canonical; designed OG image; structured data where true | Screenshot OG; missing canonical |
| D14 | **Conversion** (marketing) | One primary action; proof beside its claim; objections answered; final CTA mirrors hero; analytics on CTA | Testimonial graveyard; competing CTAs |
| D15 | **Craft detail** | Optical alignment; tabular numerals where values change; consistent icon set/weight; custom 404 and focus styles; favicon set | Mixed icon weights; default focus ring removed |

---

## Evidence rules

- A score without evidence is not a score. Code-read-only scores are labelled `(code-read)`.
- Before and after use the **same** viewport widths, themes, device profile, and network throttle.
- Perf numbers state the conditions (device, throttle, lab vs field). "Faster" without a baseline is
  not a result.
- If a dimension cannot be judged (no running app, no data), it is `Unverified` in the ledger — not 2.
