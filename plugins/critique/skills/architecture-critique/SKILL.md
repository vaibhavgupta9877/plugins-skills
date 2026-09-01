---
name: architecture-critique
description: Audit a system or module's structure — boundaries, coupling, data flow, state ownership, scalability. Use for whole-design review. Do NOT use for PR diffs — that's core-eng/code-review.
---

# architecture-critique

## Owns
- Structural review: boundaries, coupling and cohesion, layering violations.
- State ownership and data flow direction.
- Extensibility and replaceability, judged against the actual scale.

## Does not own
- Diff and PR review → `core-eng/code-review`.
- Public API shape and ergonomics → `api-critique`.
- Security and attack surface → `secure-ship/threat-model`.
- Building the restructure → the relevant stack plugin.

**Findings only. No diff required, and no code written.**

---

## Output contract

Findings ranked `blocking` / `should-fix` / `nit`, each with **what · where · why it matters ·
concrete fix**. End with the single most important change. Do not manufacture findings; a
well-structured codebase reviewed honestly gets a short report.

---

## Step 1 — Map the actual dependency direction before judging

Do not read the architecture doc and review that. Read the imports.

For each layer or module, list what it imports and what imports it. A directory named `domain` that
imports `next/navigation`, or a `data` layer that imports a widget, tells you more than any diagram.

**Most architecture problems are direction problems.** The two that recur:
- UI reaching down into data — a component importing the ORM client, a widget calling an HTTP client.
- A domain layer importing a framework — the part that is supposed to be portable is the part that
  is pinned to Next.js, Flutter, or Tokio.

Produce the real dependency picture first, in three or four lines. Every later finding anchors to it.

---

## Step 2 — Check order

1. **Are the boundaries real or nominal?** A folder named `domain` is nominal. A boundary is real
   when crossing it requires a type conversion, an interface, or a module edge the compiler enforces.
   Nominal boundaries cost the same as real ones in navigation and give none of the benefit — say so.
2. **Does state have exactly one owner?** For every meaningful piece of state, name the single place
   that owns it. Two owners is the bug behind most "why is this out of sync" behaviour. Derived state
   stored rather than computed is the same defect wearing a different hat.
3. **Is data flow unidirectional, or does it loop?** Loops (a child mutating a parent's state that
   re-renders the child) are where non-deterministic bugs live.
4. **What would it cost to replace any one layer?** Pick the DB, the HTTP client, or the state
   library and trace what would have to change. If the answer is "everywhere," the boundary is not
   doing its job. This is a measurement, not a demand that everything be replaceable.
5. **Where does business logic actually live?** Usually smeared across the UI. Name three concrete
   rules and say which file each is in. If a rule is implemented twice in two components, that is the
   finding — with both locations.

---

## Step 3 — Per-stack failure patterns

Check the ones that apply. These recur often enough to look for by name.

**Flutter**
- Business logic inside widgets; a 1500-line widget with `setState` doing coordination.
- `BuildContext` leaking out of the presentation layer into domain or data code.
- A state management approach chosen but not followed consistently — half Riverpod, half
  `ChangeNotifier`, some `setState`. **Mixed approaches are worse than any single choice.**
- A repository layer that is a thin pass-through to an HTTP client, adding a file and no value.
- DTOs escaping the data layer, so JSON field names appear in widget code.
- Navigation state scattered rather than owned by the router.

**Next.js T3**
- Server/client boundary blurred: `"use client"` at the top of a page, so the whole tree ships to the
  browser and the architecture's main benefit is gone.
- Prisma calls in components, or an ORM import reachable from a client module.
- tRPC routers mirroring database tables one-to-one instead of expressing use-cases.
- A `utils.ts` or `lib/` dumping ground — the reliable sign that no one owns placement decisions.
- Data fetched in a client component that a server component could have passed as props.
- Feature logic split across `components/`, `hooks/`, and `utils/` by *type* rather than by feature,
  so one change touches four directories.

**Rust**
- Over-generic APIs with lifetime parameters nobody needs; generics added for a second implementor
  that never arrived.
- `pub` on everything, so the internal structure is now a compatibility commitment.
- Trait abstractions with exactly one implementor — indirection with no payoff.
- `lib.rs` as an implementation dump rather than a table of contents and re-export surface.
- Error types that force the caller to match on internals, or `anyhow` in a library's public API.
- A workspace split for tidiness, creating version-lockstep pain with no release-cadence benefit.

---

## Step 4 — Judge against the actual scale

**Over-architecture for a scale that will never arrive is a finding, and a common one.** Three
abstraction layers, a plugin system, and an event bus for an app with 200 users and one developer is
a cost paid every day for an option never exercised.

Ask directly: how many users, how many developers, how many features per month, what actually changes
often? Then judge the structure against those numbers rather than against a textbook.

The inverse finding is equally real: a genuine seam missing where the codebase already changes
weekly. Point at the change history as evidence, not at a principle.

Note where a structure is right for today and will break at a specific, nameable threshold — "this
holds until you have a second writer service" is more useful than "this doesn't scale."

---

## Step 5 — Report

Rank findings by leverage: impact ÷ disruption. A boundary violation in code that never changes is a
nit; the same violation in the file touched every week is a `should-fix`.

**Always answer this explicitly: what is the one structural change with the highest
payoff-to-disruption ratio?** Give the change, the files it touches, roughly what it costs, and what
stops hurting once it lands. That single paragraph is the deliverable — everything above it is the
evidence.

If the architecture is sound for its scale, say so plainly and name the one thing to watch as it
grows.
