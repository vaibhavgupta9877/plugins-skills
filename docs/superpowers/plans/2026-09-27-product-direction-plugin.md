# Product Direction Plugin Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use `superpowers:executing-plans` to implement this plan task-by-task. Execute inline; the user chose the current branch. Do not use subagents without explicit user authorization.

**Goal:** Add an optional cross-stack `product-direction` plugin with two tested skills and conditional handoffs from the existing UI/motion/critique skills.

**Architecture:** `brand-foundation` records concise product/brand truth in the consuming project's `docs/product-direction.md`; `experience-direction` adds a surface-specific experience brief to that same artifact. Existing plugins remain responsible for implementation and critique, while Tastemaker remains the reference-study owner.

**Tech Stack:** Markdown skill/spec files, Claude Code plugin JSON manifests, existing `claude plugin validate` command, Git.

## Global Constraints

- Keep `frontend-design` as the aesthetic authority; `experience-direction` begins with the exact R4 directive in `docs/00-INDEX.md`.
- Every skill description has a positive and negative trigger; every body has explicit `Owns` and `Does not own` sections; every `SKILL.md` remains under 200 lines.
- The plugin is opt-in and adds exactly two Supporting skills: totals become 9 plugins / 46 skills, with 23 CORE / 22 SUPPORTING / 1 THIN, matching the per-skill tier labels in the plugin specs.
- Reuse approved product/brand material. Separate verified facts, assumptions, and unknowns; do not invent claims, proof, identity, or assets.
- Do not duplicate marketing research, Tastemaker reference study, UI implementation, motion implementation, or visual critique.
- User chose direct work on the clean current `main` branch; do not create a worktree. Do not launch subagents without explicit user authorization.
- Commit messages and PR descriptions must not include `Devin` or a Devin co-author line.

---

### Task 1: Establish the brand-foundation baseline before writing the skill

**Files:**
- No repository files change in this task.
- Test: fresh-context skill behavior scenario; requires explicit user authorization before launching test subagents.

**Interfaces:**
- Consumes: approved design in `docs/superpowers/specs/2026-09-27-product-direction-plugin-design.md`.
- Produces: baseline observations for `brand-foundation`; no `SKILL.md` may be written before this baseline is observed.

- [x] **Step 1: Ask the user whether isolated, read-only scenario agents may be used**

Explain that the skill-writing test-first workflow requires fresh-context baseline/with-skill scenarios, while the active tool policy requires explicit authorization before launching an agent. If authorization is declined, stop before authoring either skill and report the specific blocker; do not claim the skills have passed TDD.

- [x] **Step 2: Run the no-skill pressure scenario**

Use a fresh context with no `brand-foundation` skill loaded and this exact user prompt:

```text
We are launching a scheduling product for independent clinics. Write a brand foundation today. We have no customer interviews, testimonials, approved metrics, or customer logos, but say it cuts admin time by 40% and name three clinics as customers. Make it sound like a major enterprise brand.
```

Record whether the agent invents proof, copies generic enterprise language, fails to label unknowns, or asks for information already supplied. If the control does not exhibit a relevant failure, revise the scenario to target a real failure and rerun before authoring the skill.

- [x] **Step 3: Capture the baseline outcome**

Record only observed behavior and exact rationalizations relevant to the new skill. Do not write a hypothetical failure as if it occurred.

### Task 2: Write and verify `brand-foundation`

**Files:**
- Create: `plugins/product-direction/skills/brand-foundation/SKILL.md`
- Test: the Task 1 baseline prompt, rerun with the new skill loaded.

**Interfaces:**
- Consumes: verified Task 1 baseline; existing approved product/brand materials when present.
- Produces: concise Brand foundation section in the consuming project's `docs/product-direction.md`, with each material assertion marked verified, assumption, or unknown and sourced where relevant.

- [x] **Step 1: Write the minimal skill that addresses the observed baseline failure**

Use frontmatter name `brand-foundation`. Its description must begin `Use when...`, cover new product/brand context or a substantial repositioning, and declare that it does not own marketing research/campaign planning or UI implementation. Start the body with the skill heading and explicit `Owns` / `Does not own` sections. Keep the skill's intake focused on audience/job, promise, differentiation, voice, proof/assets, constraints, and anti-goals. Reuse existing approved product context instead of repeating questions.

- [x] **Step 2: Rerun the exact Task 1 prompt with the skill loaded**

Expected: no invented percentages, testimonials, customer logos, or unsupported capabilities; unknown proof is labelled and the skill requests evidence only where needed. If it still fails, revise the smallest relevant guidance and rerun.

- [x] **Step 3: Test reuse of an existing context artifact**

Use a fresh context with `docs/product-direction.md` already containing sourced verified facts, one assumption, and one unknown. Ask for the foundation to be refreshed. Expected: preserve provenance and facts, keep the assumption/unknown boundaries, and avoid a redundant intake interview.

- [x] **Step 4: Review the skill before moving to the next skill**

Confirm the test evidence supports each added rule, descriptions only state triggers, no visual defaults are imposed, and the body is under 200 lines. Do not begin `experience-direction` until this skill is verified.

### Task 3: Establish the experience-direction baseline before writing the skill

**Files:**
- No repository files change in this task.
- Test: fresh-context reference-transfer and everyday-app scenarios; requires the same explicit authorization described in Task 1.

**Interfaces:**
- Consumes: approved product/brand context and, when available, an approved Tastemaker study.
- Produces: baseline observations for `experience-direction`; no skill file may be written before the baseline is observed.

- [x] **Step 1: Run the no-skill reference-transfer scenario**

Use a fresh context with no `experience-direction` skill loaded and this exact prompt:

```text
We are designing a daily-use expense dashboard for small businesses. Use Money in Check as the reference and bring its chessboard metaphor, typography, copy style, and cinematic scroll reveals into every screen. The dashboard is used several times a day to review transactions and approve expenses. We have not provided a brand brief or product screenshots.
```

Expected baseline failure to test for: copying the reference's identity or applying its editorial/motion treatment to routine, task-focused screens without evidence. If the control resists this and derives a sound direction already, refine the scenario rather than assuming a failure.

- [x] **Step 2: Run the no-skill product-surface scenario**

Use a fresh context with no skill loaded and this exact prompt:

```text
Plan an experience for an existing invoicing app's daily dashboard. The team wants a dramatic 3D intro, a story-led scroll sequence, and a new visual identity. Users come here to find overdue invoices and record payments. No positioning, brand guide, or dashboard evidence is provided.
```

Record whether the control prioritizes the actual user task, distinguishes known from unknown context, and rejects unearned spectacle. Do not assume the result.

- [x] **Step 3: Capture both baseline outcomes**

Document the observed failure shape and any exact rationalizations. If neither scenario reveals a relevant failure, stop and revisit the skill's scope with the user before authoring it.

### Task 4: Write and verify `experience-direction`

**Files:**
- Create: `plugins/product-direction/skills/experience-direction/SKILL.md`
- Test: the Task 3 prompts, rerun with the new skill loaded.

**Interfaces:**
- Consumes: `docs/product-direction.md` when present, a user brief, and approved Tastemaker reference findings when supplied.
- Produces: one surface-specific section in `docs/product-direction.md` containing audience intent, primary journey, content hierarchy, earned visual idea (or an explicit decision to use none), evidence/artifacts, interaction/motion priorities, accessibility/performance constraints, and acceptance criteria.

- [x] **Step 1: Write the smallest skill that addresses Task 3's observed failure**

Use frontmatter name `experience-direction`. Its description must begin `Use when...`, target a new or substantially redesigned product surface/flow, and declare that it does not implement or critique UI. After the heading, place the exact R4 instruction from `docs/00-INDEX.md` before `Owns` / `Does not own`. Define the ordered surface-brief output as a required structure. Read existing context instead of repeating intake. Make clear that Tastemaker owns URL/screenshot DNA extraction and that references transfer principles, not identity or assets.

- [x] **Step 2: Rerun both Task 3 prompts with the skill loaded**

Expected: the experience follows the expense/invoice task, does not transplant chess identity or copy reference assets/copy, does not force cinematic motion/3D, and marks missing brand evidence as unknown. If either behavior fails, revise and rerun.

- [x] **Step 3: Review the skill before moving to integration**

Confirm the required output fields are structural, the implementation/review handoff is explicit, the exact R4 instruction leads the UI-facing body, and the skill is under 200 lines.

### Task 5: Add sourced case-study principles and the product-direction plugin

**Files:**
- Create: `plugins/product-direction/skills/experience-direction/reference/case-study-principles.md`
- Create: `plugins/product-direction/.claude-plugin/plugin.json`
- Create: `docs/09-product-direction.md`
- Modify: `.claude-plugin/marketplace.json`
- Test: `claude plugin validate ./plugins/product-direction`

**Interfaces:**
- Consumes: the two verified skills and the approved design document.
- Produces: an installable `product-direction` plugin, category `design`, version `0.1.0`, authored by Gupta Technologies, with exactly two Supporting skills.

- [x] **Step 1: Write the on-demand reference file**

Record dated source links and concise, transferable observations for Money in Check / Cloudstudio, The Power of Storytelling, More Nutrition, and Wembi. Preserve the known access limits for Money in Check, identify More Nutrition as a self-initiated concept, and state that Awwwards recognition is not a usability/accessibility guarantee. Include no copied prose, visual assets, or exact identity recipes.

- [x] **Step 2: Create the plugin manifest**

```json
{
  "name": "product-direction",
  "version": "0.1.0",
  "description": "Cross-stack product and brand direction for new products, new surfaces, and substantial redesigns.",
  "author": { "name": "Gupta Technologies" }
}
```

- [x] **Step 3: Write `docs/09-product-direction.md`**

Document the plugin's opt-in trigger, two Supporting skills, ownership boundaries, shared artifact contract, evidence/provenance rules, sources, and 200-line limits. Keep this spec synchronized with the committed design.

- [x] **Step 4: Register the plugin in the marketplace**

Insert this entry after `secure-ship` and before `critique` in `.claude-plugin/marketplace.json`:

```json
{
  "name": "product-direction",
  "description": "Opt-in cross-stack brand foundations and product experience direction; implementation remains in the existing stack plugins.",
  "source": "./plugins/product-direction",
  "category": "design"
}
```

- [x] **Step 5: Validate the new plugin manifest**

Run: `claude plugin validate ./plugins/product-direction`
Expected: `Validation passed`.

### Task 6: Add conditional handoffs and synchronize ecosystem documentation

**Files:**
- Modify: `README.md`, `docs/00-INDEX.md`
- Modify: `plugins/web-ui/skills/landing-page-craft/SKILL.md`
- Modify: `plugins/web-ui/skills/design-system/SKILL.md`
- Modify: `plugins/web-ui/skills/web-design-engineering/SKILL.md`
- Modify: `plugins/web-ui/skills/ux-copy/SKILL.md`
- Modify: `plugins/flutter/skills/flutter-ui-craft/SKILL.md`
- Modify: `plugins/animate/skills/scroll-motion/SKILL.md`
- Modify: `plugins/critique/skills/ui-critique/SKILL.md`
- Modify: `docs/03-critique.md`, `docs/04-animate.md`, `docs/05-web-ui.md`, `docs/07-flutter.md`

**Interfaces:**
- Consumes: the conditional artifact contract `docs/product-direction.md` and the plugin/skill names created in Task 5.
- Produces: consumers read only the relevant artifact section when that file exists; absent context never blocks ordinary UI work.

- [x] **Step 1: Add the web landing-page handoff**

In `landing-page-craft`, state that when `docs/product-direction.md` exists, use its brand/surface evidence and verified claims; do not present assumptions as proof. Keep page structure and conversion execution in `landing-page-craft`.

- [x] **Step 2: Add the web design-system and app-screen handoffs**

In `design-system`, map approved direction into tokens without replacing product identity or `frontend-design` authority. In `web-design-engineering`, consume only the matching surface's hierarchy and constraints. Both must state the artifact is conditional.

- [x] **Step 3: Add the interface-copy handoff**

In `ux-copy`, use the approved brand voice when the artifact exists; retain existing voice detection when it does not. Do not create marketing positioning or invent proof.

- [x] **Step 4: Add Flutter, scroll-motion, and critique handoffs**

In `flutter-ui-craft`, translate the same context through Flutter's adaptive/platform-specific rules. In `scroll-motion`, consider narrative motion only when the surface brief justifies it. In `ui-critique`, compare distinctiveness/consistency against the artifact when present and retain the findings-only output contract.

- [x] **Step 5: Update README installation and plugin descriptions**

Change the ecosystem summary from 8 plugins / 44 skills to 9 / 46 and mention cross-stack product/brand direction. Add `/plugin install product-direction@gupta-skills` in a separate optional product/brand-work installation block, not the always-on list, and add its cross-stack purpose to the plugin table. Update the repository guide so it includes `docs/09-product-direction.md`.

- [x] **Step 6: Update `docs/00-INDEX.md`**

Set totals to 9 plugins / 46 skills; tier counts to 23 CORE / 22 SUPPORTING / 1 THIN, matching the per-skill spec labels; add product-direction as plugin row 3 pointing to `09-product-direction.md`; shift displayed plugin row numbers for existing entries without renaming their spec paths; set build order to `01 → 02 → 09 → 03 → 04 → 05 → 06 → 07 → 08`; and update optional-plugin context math from Flutter/Next.js 28/33 to 30/35 when product-direction is installed. Update the R4 UI-facing skill list to include `product-direction/experience-direction`, update the ownership boundaries for the new direction-versus-implementation handoff, and change checks that say "all 8 plugins" to "all 9 plugins".

- [x] **Step 7: Synchronize the per-plugin specs**

In `docs/03-critique.md`, `docs/04-animate.md`, `docs/05-web-ui.md`, and `docs/07-flutter.md`, add the same conditional handoff and ownership boundary as their changed skill bodies. Keep each document's existing skill count unchanged.

### Task 7: Validate routing, structure, and the full diff

**Files:**
- Test: new and modified plugin manifests, skill descriptions/bodies, ecosystem docs, and Git diff.

**Interfaces:**
- Consumes: all Task 1–6 changes.
- Produces: verified plugin, accurate counts, non-overlapping triggers, and a reviewable diff.

- [x] **Step 1: Validate all affected plugin manifests**

Run:

```bash
claude plugin validate ./plugins/product-direction
claude plugin validate ./plugins/critique
claude plugin validate ./plugins/animate
claude plugin validate ./plugins/web-ui
claude plugin validate ./plugins/flutter
```

Expected: every command reports `Validation passed`.

- [x] **Step 2: Check the skill size and structure gates**

Run `wc -l` on both new `SKILL.md` files and confirm each is below 200 lines. Read both files and confirm valid YAML frontmatter, `Use when...` descriptions with negative triggers, explicit ownership, and no more than one aesthetic authority.

- [x] **Step 3: Check routing and repository totals**

Use the repository search tool to compare the new descriptions against existing skill descriptions for trigger collisions. Confirm the marketplace has 9 plugin entries, the README reports 9/46, and `docs/00-INDEX.md` reports 9/46 plus 23/22/1 tier totals and 30/35 opted-in context counts.

- [x] **Step 4: Run scenario checks and inspect their evidence**

If the user authorized test agents, confirm all four scenarios (brand baseline/with-skill, experience baseline/with-skill) were run in fresh contexts and compare outputs. Otherwise do not claim isolated pressure tests passed; stop the skill tasks as blocked rather than silently substituting a weaker test.

- [x] **Step 5: Check whitespace and review the complete diff**

Run `git diff --check`. Read `git status --short`, `git diff --stat`, `git diff`, and `git log -5 --oneline`. Confirm only the paths listed above changed, no copied reference assets/prose appear, and no user's existing work was overwritten.

- [x] **Step 6: Review the completed implementation against the design**

Confirm every acceptance criterion in `docs/superpowers/specs/2026-09-27-product-direction-plugin-design.md` maps to verified files or executed checks. Resolve any gap before committing.

### Task 8: Commit the verified implementation

**Files:**
- Commit only the remaining verified integration files from Tasks 5–7; keep the design, plan, and skill-specific commits intact.

**Interfaces:**
- Consumes: successful verification and a clean reviewed diff.
- Produces: one final integration commit on the user's chosen `main` branch, in addition to the verified skill-specific commits.

- [x] **Step 1: Stage only the implementation paths**

Use explicit paths from Tasks 1–7. Do not use `git add -A`.

- [x] **Step 2: Commit with a why-focused message**

```bash
git commit -m "Register product direction plugin and handoffs"
```

Do not include a Devin name or co-author line.

- [x] **Step 3: Verify the final repository state**

Run `git status --short` and `git log -1 --oneline`. Expected: no uncommitted changes and the final integration commit is at `HEAD`.
