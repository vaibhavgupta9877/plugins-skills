---
name: api-critique
description: Review a public API surface — REST/tRPC endpoints, Rust crate public items, Dart package exports — for ergonomics, consistency, and evolvability. Do NOT use for internal implementation review.
---

# api-critique

## Owns
- Public surface review: naming consistency, error contracts, evolvability.
- Breaking-change detection against a previous version.
- What is exported that should not be.

## Does not own
- Designing a new API → `next-saas/api-design`, `rust-crate/api-ergonomics`.
- Internal structure and layering → `architecture-critique`.
- Implementation correctness → `core-eng/code-review`.
- Auth and abuse → `secure-ship/threat-model`.

**Findings only. No code written.**

---

## Output contract

Findings ranked `blocking` / `should-fix` / `nit`, each with **what · where · why it matters ·
concrete fix**. End with the single most important change.

For a published surface, `blocking` means "shipping this locks in a mistake you cannot take back."
That bar is higher than for internal code, and it is the whole reason this skill exists.

---

## Step 1 — The test that matters

**Can a competent stranger use this correctly on the first attempt, using only the type signatures
and the docs?**

Run it literally. Take the three most common operations, write the call site you would write with no
knowledge of the implementation, and see where you would guess wrong. Every place you had to open the
source to know what to pass, what comes back, or what happens on failure is a finding.

If a README or doc example exists, judge that example as the API's own claim about itself. An awkward
example means an awkward API — the fix is the API, not the example.

---

## Step 2 — Check order

1. **Naming consistency across the whole surface.** One concept, one word, everywhere. Mixed
   `fetch`/`get`/`load`, or `remove` in one place and `delete` in another, is a real defect: it makes
   the surface unguessable and therefore unmemorable. Check parameter order consistency too.
2. **Is the error contract uniform and typed?** One shape for every failure, discoverable from the
   type system. A surface where some calls throw, some return null, and some return an error object
   forces defensive code at every call site.
3. **Are the optional/required boundaries honest?** Optional parameters that are actually required in
   half the cases; required parameters with an obvious default; a config object where three fields
   are mutually exclusive. Encode the real constraint in the type where the language allows it.
4. **Is anything exported that should not be?** Internal types leaked through a return value, a
   helper made public for testing, a module re-exported wholesale.
5. **What becomes a breaking change later?** Walk the plausible next three features and ask which
   would force a major version. That is the cost of today's shape.

---

## Step 3 — Rust public crates

Every `pub` item is a permanent commitment. Flag specifically:

- **Leaked internal types** — a public function returning a type the consumer cannot name, construct,
  or match on, or one from a private module.
- **Missing `#[non_exhaustive]`** on any enum or struct likely to grow. Without it, adding a variant
  or a field is a breaking change forever.
- **Generic parameters that force turbofish at call sites.** If the example needs `::<_, _>` to
  compile, the signature is wrong for its callers. Over-generic APIs are a common ergonomics failure.
- **Error types that do not implement `std::error::Error`**, or that break the `source()` chain, or
  that expose internals the caller must match on.
- **Missing standard derives.** A public type without `Debug` is hostile to every consumer debugging
  their own code. Then check `Clone`, `Default`, `PartialEq`, and `Serialize`/`Deserialize` behind a
  feature.
- **A dependency's type in the public signature** — that version-locks your consumers to that
  dependency's major version whether they wanted it or not.
- **Naming against the Rust API Guidelines.** `as_`/`to_`/`into_` have specific meanings
  (borrowed→borrowed, expensive conversion, owned→owned); `iter`/`iter_mut`/`into_iter`; getters
  without a `get_` prefix. Ecosystem consistency beats local preference.
- **Undocumented panics.** A public function that can panic and does not say so is a bug.

## Step 4 — tRPC

- **Procedures should express use-cases, not CRUD over tables.** A router that mirrors the Prisma
  schema one-to-one pushes orchestration into the client, where it will be implemented twice and
  differently.
- **Input schemas that accept more than the procedure uses** — a pass-through object, a `z.any()`, or
  a field the handler ignores. Each is an invitation to send something unexpected.
- **An input that accepts a tenant or org id from the client.** Flag this as `blocking` and route it
  to `secure-ship/threat-model` and `next-saas/trpc-prisma-patterns` — it is the IDOR shape.
- **Mutations that return nothing useful.** Returning the updated entity saves the client a
  round-trip and an invalidation.
- Error codes used consistently: `UNAUTHORIZED` vs `FORBIDDEN` vs `NOT_FOUND` mean different things
  and drive different client behaviour.
- Pagination shape consistent across the surface, and cursor-based rather than offset.

## Step 5 — Dart package exports

- A single `lib/<package>.dart` barrel that exports exactly the intended surface; everything else
  under `lib/src/`. Anything importable from `lib/src/` directly is public by accident.
- Sealed classes or enums that will grow, exported without a documented evolution story.
- `dynamic` in a public signature — it removes the only guarantee the consumer had.

---

## Step 6 — Breaking-change detection

When a previous version exists, diff the surface explicitly rather than judging from memory. Say
which version you compared against.

Remember what counts as breaking beyond the obvious signature change:
- Adding an enum variant, or a public struct field, without `#[non_exhaustive]`.
- Adding a trait method with no default implementation.
- Tightening a bound, or narrowing an accepted type.
- Bumping the major version of a dependency whose types appear in your signatures.
- Changing an error variant, a status code, or a serialized field name.
- Making a previously-optional field required — including in a Zod input schema.

For Rust, `cargo semver-checks` and `cargo public-api` catch the mechanical cases; you own the
semantic ones (a function that now returns fewer results, an error that now fires in a new case).

---

## Step 7 — Report

Anchor findings to the item, not the file: `pub fn Client::connect`, `invoice.list`, `export
FooOptions`.

Close with the single most important change — and where a version is about to be published, say
plainly whether it should be published as-is, because after publication most of these findings become
permanent.
