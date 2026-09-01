---
name: spec-writer
description: Turn locked requirements into a phased implementation spec or task plan with acceptance criteria. Use after grill-me. Do NOT use to gather requirements — that's grill-me.
---

# spec-writer

## Owns
- Spec and plan documents.
- Phasing, task decomposition, sequencing, dependencies.
- Acceptance criteria and done-conditions.
- Relative complexity estimates.

## Does not own
- Requirement gathering → `grill-me`.
- Reviewing the code that results → `code-review`.
- Test authoring → `test-strategy`.

---

## Precondition

Requirements must already be locked. If you cannot state the goal, the non-goals and the
acceptance criteria without hedging, stop and run `grill-me` first. Writing a spec on
unlocked requirements produces a confident document about the wrong thing.

---

## Output

Markdown only, unless the user names another format. Default filename `PlannedTasks.md`
in the repo root; use the repo's existing convention if one exists (`docs/plan.md`,
`SPEC.md`, an issue tracker template).

---

## Structure

```markdown
# <Project / feature name>

## Goal
One paragraph. What is true when this is done.

## Non-goals
Bulleted. Explicitly out of scope. Carried over from grill-me verbatim.

## Constraints
Stack, versions, deploy target, compatibility, deadline, team size.

## Open decisions
Anything requiring a call the user has not made. Each with the options and your
recommendation. Do not silently pick.

## Phase 1 — <name>   [S/M/L]
Goal of this phase. What ships.

- **P1-001** — <task title>
  - What: one sentence.
  - Done when: an observable condition.
  - Depends on: — / P1-000
  - Effort: S

## Phase 2 — ...

## Risks
What could invalidate the plan, and the earliest signal you would see it.
```

---

## Rules

**Phases must be independently shippable.** A phase that cannot be merged and deployed on
its own is mis-cut. Test each phase against: *if we stopped after this one, is the product
in a coherent, deployable state?* If no, re-cut. "Backend" and "frontend" are almost never
valid phases; "user can sign in" is.

**Every task states its done-condition.** "Implement auth" is not a task. "User can sign in
with email + password and the session survives a page refresh" is. The done-condition must
be observable — by a test, a user action, or a command with an expected output.

**Tasks are atomic.** One deliverable, completable in one focused session. If a task needs
three unrelated files changed for three unrelated reasons, it is three tasks.

**Number tasks** `P<phase>-<nnn>`. Stable IDs let the user talk about the plan later.

**Estimate in relative complexity — S / M / L. Never hours.** S: understood, mechanical.
M: some design work or unfamiliar surface. L: contains unknowns — and an L task should
usually be split, or preceded by a spike task that resolves the unknown.

**Flag tasks that carry a decision or a dependency:**
- `[DESIGN]` — needs a visual/UX decision before it can start.
- `[BACKEND]` / `[INFRA]` — needs a service, migration, or environment that may not exist.
- `[DECISION]` — blocked on the user; must also appear under Open decisions.
- `[SPIKE]` — timeboxed investigation whose output is knowledge, not shipped code.

**Flag anything requiring a decision the user has not made.** Do not silently pick a
database, a payment provider, or an auth model and bury it in task P2-004. Surface it in
Open decisions with your recommendation, and mark the dependent tasks `[DECISION]`.

**Sequence by dependency and by risk.** Front-load the tasks that could invalidate the
plan. If the data model might be wrong, that is Phase 1, not Phase 3.

**Test tasks are not a phase.** Testing is part of each task's done-condition. A phase
called "Testing" at the end means the earlier phases were never actually done. Consult
`test-strategy` for which layer each task's verification belongs at.

---

## Detecting the repo's conventions

Before writing, check whether the repo already has a plan format — an existing
`PlannedTasks.md`, `docs/roadmap.md`, or issue templates. Match it. A second competing
plan format is worse than a slightly wrong one.

---

## Handing off

Close the document with a one-line next action: which task to start, and which skill owns
it. If the plan contains `[DECISION]` items in Phase 1, say plainly that Phase 1 is blocked
until those are answered.
