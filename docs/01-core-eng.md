# Plugin: `core-eng`

**Enable for:** every project, always.
**Purpose:** the discipline layer. Extracting real requirements, planning, reviewing diffs, testing, debugging. Stack-agnostic.
**Skills:** 6

---

## `grill-me` — CORE

**description:** `Interrogate the user with interactive multiple-choice questions before any build, refactor, or architecture decision, until requirements are unambiguous. Do NOT use for quick factual questions or one-line fixes.`

**Owns:** requirement extraction, ambiguity elimination, scope negotiation, feasibility pushback.
**Does NOT own:** writing the spec document → `spec-writer`.

**Body must cover:**
- This is the **entry point**. Any request that will produce more than ~50 lines of code, or that names a feature rather than a fix, triggers this before anything else.
- Use the interactive question tool where available (tappable options, max 3 questions per round, 2–4 mutually-exclusive options each). On CLI, numbered lists.
- Question selection heuristic: ask only what **cannot be derived** from the codebase, existing skills, or the conversation. Read the repo first. Asking about something visible in `package.json` destroys trust.
- Round structure: state 2–4 blunt observations about the request first (what's wrong, what already exists, what's infeasible), *then* ask. Informed answers beat cold ones.
- Stop condition: continue rounds until you can restate the requirement, the non-goals, and the acceptance criteria without hedging. Typically 2–3 rounds. Never exceed 4 — at that point the ambiguity is the user's to resolve in prose.
- **Mandatory pushback.** Do not accept a request because it was asked for. Flag: work that duplicates something already built; scope that won't pay for itself; architecture the team can't maintain; features that need infrastructure that doesn't exist yet. Say "this is not worth building" when it is not worth building.
- Always ask about non-goals explicitly. What the user does *not* want is higher-signal than what they do.
- Close each round by summarising what is now locked, so decisions accumulate visibly.

**reference/:** `question-patterns.md` — reusable question sets per work type (new feature, refactor, new project, bug, integration, redesign).

**Budget:** 180 lines.

---

## `spec-writer` — CORE

**description:** `Turn locked requirements into a phased implementation spec or task plan with acceptance criteria. Use after grill-me. Do NOT use to gather requirements — that's grill-me.`

**Owns:** spec/plan documents, phasing, task decomposition, acceptance criteria, sequencing.
**Does NOT own:** requirement gathering → `grill-me`; code review → `code-review`.

**Body must cover:**
- Output format: markdown only, unless the user names another format.
- Structure: Goal · Non-goals · Constraints · Phases · Per-phase tasks · Acceptance criteria · Open risks.
- Phases must be independently shippable. A phase that can't be merged and deployed alone is mis-cut.
- Every task states its done-condition. "Implement auth" is not a task. "User can sign in with email+password and the session survives a refresh" is.
- Estimate in relative complexity (S/M/L), never in hours.
- Flag anything requiring a decision the user hasn't made — do not silently pick.
- **Absorb:** the `PlannedTasks.md` convention and phasing format from `/mnt/skills/user/flutter-fullstack` — generalise it beyond Flutter.

**Absorb:** `/mnt/skills/user/flutter-fullstack` (planning half only; the Flutter architecture half goes to `flutter/flutter-architecture`).

**Budget:** 150 lines.

---

## `code-review` — CORE

**description:** `Review a diff, PR, or changeset for correctness, regressions, and structural problems. Do NOT use to audit a whole system's design — that's critique/architecture-critique.`

**Owns:** diff-time review, correctness, edge cases, regression risk, review comment authoring.
**Does NOT own:** holistic architecture audits → `critique/architecture-critique`; security threat modelling → `secure-ship/threat-model`; API shape → `critique/api-critique`.

**Body must cover:**
- Read the diff *and* enough surrounding code to judge it. A diff reviewed in isolation misses the bugs that matter.
- Priority order: correctness → data loss / security regression → error handling → concurrency → performance → structure → style. Never lead with style.
- Explicit checks: unhandled error paths, silent catches, `null`/`None`/`?` misuse, off-by-one, unbounded loops or queries, N+1, missing transactions, missing input validation at trust boundaries, mutation of shared state.
- Every comment states severity (blocking / should-fix / nit) and the concrete fix. Comments without a proposed fix are noise.
- Say "this looks correct" when it does. Manufactured findings train the user to ignore reviews.
- Call out anything the diff *should* have included and didn't — tests, migration, changelog, docs.

**Budget:** 160 lines.

---

## `git-pr` — SUPPORTING

**description:** `Branch naming, commit messages, PR descriptions, and merge hygiene. Use when committing, opening a PR, or cleaning history. Do NOT use to review code content — that's code-review.`

**Owns:** commit/branch/PR conventions, history hygiene, changelog entries.
**Does NOT own:** review content → `code-review`; release tagging for crates → `rust-crate/publish-open-source`.

**Body must cover:**
- Detect the repo's existing convention from `git log` before imposing one. Consistency with the repo beats any external standard.
- Default if none exists: Conventional Commits (`feat:`, `fix:`, `chore:`, `refactor:`, `docs:`, `test:`), imperative mood, body explains *why* not *what*.
- Branch: `<type>/<short-kebab-description>`.
- PR description template: what changed · why · how to test · risk · screenshots for UI.
- Never force-push a shared branch. Never rewrite history that's been pulled.
- Public Rust crates: commits feed the changelog, so they must be written for external readers.

**Budget:** 90 lines.

---

## `test-strategy` — CORE

**description:** `Decide what to test at which layer, and write the tests. Covers unit, integration, and end-to-end across Flutter, Rust, and Next.js. Do NOT use for manual QA checklists or prod readiness — that's secure-ship/prod-readiness-gate.`

**Owns:** test layer selection, test authoring, fixtures, coverage judgement, e2e flows.
**Does NOT own:** release gating → `secure-ship/prod-readiness-gate`; performance benchmarking → `rust-crate/perf-and-unsafe`, `web-ui/web-performance`.

**Body must cover:**
- Layer decision rule: unit-test pure logic and edge cases; integration-test anything crossing a boundary (DB, FS, network, platform channel); e2e-test only the 3–7 flows whose breakage means the product is down. Do not e2e-test what a unit test can catch.
- Per-stack tooling: Flutter (`flutter_test`, `integration_test`, golden tests for UI, `mocktail`), Rust (`#[cfg(test)]` unit, `tests/` integration, `proptest` for invariants, doctests as public-API contracts), Next.js (Vitest, React Testing Library, Playwright for e2e, tRPC procedure tests).
- Coverage is a diagnostic, not a target. Chasing a percentage produces tests that assert nothing.
- Every bug fix ships with a test that fails without the fix.
- Golden/snapshot tests: valuable in Flutter for UI regressions, dangerous in web where they churn. State this distinction.
- **Absorb:** everything in `/mnt/skills/user/e2e-testing`.

**Absorb:** `/mnt/skills/user/e2e-testing`

**reference/:** `stack-recipes.md` — setup + config per stack.

**Budget:** 190 lines.

---

## `debug` — SUPPORTING

**description:** `Systematically diagnose a bug, crash, or unexpected behaviour by narrowing hypotheses against evidence. Use when something is broken and the cause is unknown. Do NOT use when the fix is already identified.`

**Owns:** root-cause analysis, hypothesis narrowing, instrumentation, reproduction.
**Does NOT own:** reviewing a proposed fix → `code-review`.

**Body must cover:**
- Reproduce first. A bug that can't be reproduced can't be confirmed fixed. If reproduction is impossible, say so and instrument instead of guessing.
- Bisect the surface: is it data, environment, timing, or logic? Each has a different narrowing strategy.
- State a falsifiable hypothesis, then design the cheapest test that kills it. Never change code hoping it helps.
- Read the actual error and stack trace fully before theorising. Most wasted debugging is skipped reading.
- Stack traps to name: Flutter (build-context misuse, async gaps after `await`, platform-channel failures on desktop), Rust (borrow/lifetime errors masking a design problem, `unwrap` in library code, async runtime mismatch), Next.js (server/client boundary, hydration mismatch, stale cache, edge-vs-node runtime).
- Once found: state root cause, blast radius, fix, and the test that pins it.

**Budget:** 130 lines.
