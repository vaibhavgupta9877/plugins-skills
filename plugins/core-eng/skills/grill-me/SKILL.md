---
name: grill-me
description: Interrogate the user with interactive multiple-choice questions before any build, refactor, or architecture decision, until requirements are unambiguous. Do NOT use for quick factual questions or one-line fixes.
---

# grill-me

## Owns
- Requirement extraction and ambiguity elimination.
- Scope negotiation and non-goal capture.
- Feasibility pushback — telling the user when something should not be built.
- Locking decisions so later skills can rely on them.

## Does not own
- Writing the spec or task plan → `spec-writer`.
- Reviewing code → `code-review`.
- Auditing an existing design → `critique/architecture-critique`.

---

## When this fires

This is the **entry point**. Trigger it before anything else when the request either:

- will produce more than roughly 50 lines of code, or
- names a *feature* rather than a *fix* ("add billing", "redo onboarding", "make it multi-tenant"), or
- implies a structural decision (new dependency, new service, new data model, new platform).

Do **not** fire for: one-line fixes, factual lookups, "what does this function do", formatting, or a task whose acceptance criteria are already written down.

If in doubt: one question is cheap, a wrong week of work is not.

---

## Step 1 — Read before you ask

Ask only what **cannot be derived**. Asking about something visible in the repo destroys trust and wastes a round.

Before the first question, spend real effort on:

- `README`, `CLAUDE.md`, `AGENTS.md`, any `docs/` or `spec/` directory.
- Manifest files — `package.json`, `Cargo.toml`, `pubspec.yaml` — for stack, versions, and what is already a dependency.
- Directory structure, to learn the conventions and what already exists.
- `git log --oneline -30` for recent direction and the team's vocabulary.
- The conversation so far. Never re-ask something the user already stated.

Derive the stack, the conventions, the existing feature surface, and the constraints. Ask about intent, priority, trade-offs, and non-goals — the things only the user knows.

---

## Step 2 — Observe, then ask

Open every round with **2–4 blunt observations** about the request, *then* the questions. Informed answers beat cold ones, and the observations are where pushback lives.

Good observations name a specific fact:

- "You already have `AuthProvider` doing session refresh — this overlaps ~70%."
- "This needs a job queue. There isn't one, and adding one is a bigger decision than this feature."
- "The repo is on Riverpod. A BLoC-shaped answer here means two state systems."

Bad observations are hedges: "there are some trade-offs to consider."

---

## Step 3 — Question mechanics

- Use the interactive question tool where available: tappable options, **max 3 questions per round**, **2–4 mutually exclusive options** each. On a plain CLI, use numbered lists and say "reply with numbers".
- Every option must be a real, distinct commitment. Never offer options that differ only in wording.
- Order options so your recommendation is first, and mark it. Do not hide behind neutrality — you have read the code, they have not.
- Never ask a question whose answer you could get by reading one more file.

See `reference/question-patterns.md` for ready-made question sets per work type.

---

## Step 4 — Mandatory pushback

Do not accept a request merely because it was asked for. In each round, explicitly check for and flag:

- **Duplication** — this already exists, in this repo or in a dependency.
- **Scope that will not pay for itself** — the maintenance cost exceeds the value.
- **Unmaintainable architecture** — the team cannot run this once you leave.
- **Missing prerequisite infrastructure** — this needs auth, a queue, observability, or a migration path that does not exist yet.
- **Sequencing errors** — this is the third thing, not the first.

Say **"this is not worth building"** when it is not worth building. Then offer the smaller thing that is.

If the user hears the objection and reaffirms the request, that is their decision: say so in one line, drop the objection, and proceed with the full scope.

---

## Step 5 — Non-goals, always

Every session asks explicitly about non-goals. What the user does *not* want is higher-signal than what they do, and it is the thing they never volunteer.

Concretely: "Which of these are explicitly **out** for this pass?" with options drawn from the adjacent work the feature invites — offline support, migration of existing data, admin UI, i18n, mobile layout, rate limiting.

---

## Step 6 — Close each round

End every round with a **Locked** block, so decisions accumulate visibly and cannot be silently reopened:

```
Locked so far
- Scope: <one line>
- Non-goals: <list>
- Constraints: <stack / deadline / compatibility>
- Still open: <what the next round resolves>
```

---

## Stop condition

Continue rounds until you can restate, **without hedging**:

1. the requirement,
2. the non-goals,
3. the acceptance criteria.

Typically 2–3 rounds. **Never exceed 4.** At round 4 the remaining ambiguity is not answerable by multiple choice — stop asking, state the residual ambiguity plainly, and ask the user to resolve it in prose.

When the stop condition is met, hand off: "Requirements locked. Next: `spec-writer` to turn this into a phased plan" — or, for small scoped work, go straight to implementation.
