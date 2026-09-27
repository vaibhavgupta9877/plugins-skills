# Skills Ecosystem — Build Specification

**Owner:** Gupta Technologies (guptatechnologies.in)
**Stacks:** Flutter (Android/Windows/Linux) · Rust crates (incl. public/open-source) · Next.js T3 SaaS (Vercel-primary) · company marketing site
**Deliverable:** 9 Claude Code plugins, 46 skills.

You are the building agent. Read this file completely before touching anything. Then build one plugin per spec file, in the order listed.

---

## 1. Non-negotiable rules

**R1 — Size budget.**
- `description:` frontmatter ≤ 2 lines / ~35 words. This is always in context. Every word costs.
- `SKILL.md` body ≤ 200 lines. Hard cap.
- Anything longer goes in `reference/*.md` inside the skill folder, loaded on demand via an explicit instruction in the body.
- If a skill body wants to exceed 200 lines, that's a signal it's two skills. Split it or cut scope.

**R2 — Every description declares negative triggers.**
Format: `Use when <X>. Do NOT use for <Y> — that's <other-skill>.`
This is the single biggest defence against the collision problem that broke the previous setup.

**R3 — Every skill body opens with an Owns / Does not own block.** Two short lists. No exceptions.

**R4 — Do not duplicate `frontend-design`.**
Every UI-facing skill (all of `web-ui`, all of `animate`, `ui-critique`, `motion-critique`, `flutter-ui-craft`, and `product-direction/experience-direction`) must open its body with:

> Before doing anything else, read `/mnt/skills/public/frontend-design/SKILL.md` and follow it. It is the source of truth for aesthetic direction, typography, palette, restraint, and interface copy. This skill adds stack-specific execution on top of it and never contradicts it.

Do not copy its rules inline. Duplicated rules drift.

**R5 — Skills encode opinions and workflows, not language reference.**
The model already knows Dart, Rust and TypeScript. A skill that explains syntax is dead weight. A skill that says "in this codebase, errors are `thiserror` enums at library boundaries and `anyhow` only in binaries" earns its slot. When you don't know the convention, the skill must instruct the agent to detect it from the codebase or ask — never invent one.

**R6 — Absorb, then delete.**
Before writing any skill marked with an `Absorb:` line, read the named existing skill in full and merge everything worth keeping. After all plugins are built and verified, delete these:
```
/mnt/skills/user/t3-frontend-craft
/mnt/skills/user/t3-next-setup
/mnt/skills/user/str-landing-setup
/mnt/skills/user/flutter-ui-design
/mnt/skills/user/flutter-fullstack
/mnt/skills/user/design-system
/mnt/skills/user/e2e-testing
```
Also disable the `design:` plugin — `critique` and `web-ui` replace it entirely.
**Do not delete anything until the replacement skill is written and you have confirmed with the user.**

**R7 — External sources are fetched, not trusted.**
Where a spec lists a source URL, fetch it and distil. Take the *concepts and heuristics*, not the prose. Do not vendor another repo's SKILL.md verbatim — licences differ and their file sizes will blow R1. Credit sources in a `## Sources` footer line.

**R8 — Cross-plugin references use paths, not copies.**
If `flutter-motion-craft` needs vocabulary from `animation-vocabulary`, it points at the path. Never inlines.

---

## 2. Plugin structure

```
<plugin-name>/
  .claude-plugin/plugin.json
  skills/
    <skill-name>/
      SKILL.md
      reference/*.md        # optional, on-demand
```

`plugin.json`:
```json
{
  "name": "flutter",
  "version": "0.1.0",
  "description": "One line. What projects this is for.",
  "author": { "name": "Gupta Technologies" }
}
```

---

## 3. Plugin map

| # | Plugin | Enable for | Skills | Spec file |
|---|--------|-----------|--------|-----------|
| 1 | `core-eng` | **always** | 6 | `01-core-eng.md` |
| 2 | `secure-ship` | **always** | 4 | `02-secure-ship.md` |
| 3 | `product-direction` | design-ready brand context or new product-surface direction, any stack | 2 | `09-product-direction.md` |
| 4 | `critique` | **always** | 6 | `03-critique.md` |
| 5 | `animate` | any UI work | 7 | `04-animate.md` |
| 6 | `web-ui` | Next.js + company site | 5 | `05-web-ui.md` |
| 7 | `next-saas` | Next.js only | 5 | `06-next-saas.md` |
| 8 | `flutter` | Flutter only | 5 | `07-flutter.md` |
| 9 | `rust-crate` | Rust only | 6 | `08-rust-crate.md` |

**Build order:** `01` → `02` → `09` → `03` → `04` → `05` → `06` → `07` → `08`. Later plugins reference earlier ones.

---

## 4. Ownership boundaries (memorise these)

These four lines resolve ~90% of potential collisions:

- **Direction vs making vs judging.** `product-direction` sets brand and surface direction; `animate`, `web-ui`, `flutter`, `next-saas`, and `rust-crate` build; `critique` judges. A skill never owns both direction and implementation/review.
- **Diff vs design.** `core-eng/code-review` reads a diff and checks correctness. `critique/*` audits a whole design or architecture with no diff required.
- **System vs craft.** `flutter/flutter-animation-system` is API knowledge (AnimationController, curves, Hero, custom painters). `animate/flutter-motion-craft` is taste — what to animate, how long, what it should feel like.
- **Always-on vs opt-in.** `core-eng`, `secure-ship`, `critique` load in every project. `product-direction` and the stack-specific plugins are opt-in per repo.

---

## 5. Honest tiering

46 skills is more than you will realistically use. Every skill in the spec files carries a tier:

- **CORE** (23 skills) — will fire constantly. Write these first, write them well, spend the full 200-line budget if needed.
- **SUPPORTING** (22 skills) — fire a few times per project. Keep to ~100 lines.
- **THIN** (1 skill) — narrow scope, may rarely fire. Keep to ~60 lines. If after 60 days the THIN skill has never fired, delete it rather than maintaining it.

Do not pretend all 46 are equal. Budget effort accordingly.

---

## 6. Context math (why R1 exists)

Only *enabled* plugins load their descriptions. Worst case per project:

- Flutter project without product-direction: 16 always-on + 7 animate + 5 flutter = **28 descriptions** ≈ 1.0k tokens; with product-direction, add 2 for **30**.
- Next.js project without product-direction: 16 + 7 + 5 + 5 = **33 descriptions** ≈ 1.2k tokens; with product-direction, add 2 for **35**.

That is acceptable. What is *not* acceptable is three 600-line skill bodies firing at once — 5k+ tokens for one question. R1 is the entire defence. Enforce it.

---

## 7. Verification before you hand back

For each plugin, confirm:
- [ ] Every `description` states both a positive and a negative trigger.
- [ ] No two skills across all 9 plugins claim the same trigger phrase.
- [ ] Every body has Owns / Does not own.
- [ ] Every UI-facing skill has the R4 `frontend-design` directive as its first instruction.
- [ ] No `SKILL.md` exceeds 200 lines. Report actual line counts in a table.
- [ ] Every `Absorb:` source was read before the new skill was written.
- [ ] Nothing deleted yet — present the deletion list to the user for approval.
