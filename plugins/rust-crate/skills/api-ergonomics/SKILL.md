---
name: api-ergonomics
description: Design the public API of a Rust crate for external consumers — naming, trait design, generics vs dyn, builders, semver-safe evolution. Do NOT use to review an existing API, that's critique/api-critique.
---

# api-ergonomics

## Owns
- Designing a public surface that strangers can use correctly on the first try.

## Does not own
- Reviewing an existing surface → `critique/api-critique`.
- Internal structure, visibility policy, error architecture → `crate-architecture`.
- Publish mechanics and semver tooling → `publish-open-source`.

**These crates are published.** Every decision here is one you will be held to by people you will
never meet, in code you cannot see. Design accordingly.

---

## 1. Write the call site first

Before the types, write the code you want someone to be able to write:

```rust
let client = Client::builder()
    .base_url("https://api.example.com")
    .timeout(Duration::from_secs(10))
    .build()?;

let invoices = client.invoices().list().status(Status::Overdue).send().await?;
```

Then build the types that make it true.

**If the example in the README is awkward, the API is wrong — fix the API, not the example.** The
temptation to add a sentence of explanation instead of removing a parameter is the single most
reliable sign a surface is about to age badly. Every consumer pays that awkwardness forever; you pay
the redesign once, now, while nobody depends on it.

Write three call sites, not one: the trivial case, the configured case, and the awkward case you
know someone will hit. If the trivial case needs more than one line of setup, the defaults are wrong.

---

## 2. Follow the ecosystem's naming, not your own

Rust has established conventions and users read them as meaning. Consistency with the ecosystem is
worth more than any local preference — see `reference/guidelines-checklist.md`.

| Prefix | Means | Cost |
|---|---|---|
| `as_` | borrowed → borrowed, free | No allocation, no failure |
| `to_` | expensive conversion | Allocates or computes |
| `into_` | owned → owned, consuming | Takes `self` |

- `iter()` / `iter_mut()` / `into_iter()` — the three forms, named exactly this.
- **Getters have no `get_` prefix**: `fn name(&self) -> &str`, not `get_name`. `get_` is reserved for
  the fallible-indexing sense (`slice::get`).
- `is_` / `has_` for predicates returning `bool`.
- `try_` for the fallible variant of an infallible operation.
- `new` is the obvious constructor; `with_*` for a variant; a builder once there are more than three
  or four knobs.
- Types are `UpperCamelCase`, and a type's name should not repeat its module: `error::Error`, not
  `error::CrateError`.

**Naming against convention is not a style disagreement — it makes the crate feel foreign** and
forces every user to check the docs for something they already knew.

---

## 3. Accept generously, return concretely

```rust
// Accept
pub fn open(path: impl AsRef<Path>) -> Result<File>;
pub fn extend(&mut self, items: impl IntoIterator<Item = Row>);
pub fn write_all(&mut self, buf: &[u8]) -> Result<()>;

// Return
pub fn rows(&self) -> impl Iterator<Item = &Row> + '_;   // not Box<dyn Iterator>
pub fn name(&self) -> &str;                              // not String
```

- Take `&str`/`&[T]`/`&Path` over owned types unless you genuinely need ownership; `impl AsRef<_>`
  when both are plausible at the call site.
- **Never take `String` when `&str` will do** — it forces every caller to clone.
- Return concrete types or `impl Trait`. `Box<dyn Trait>` in a return position where a concrete type
  works costs an allocation, blocks inlining, and erases traits the caller might have wanted.
- **Do not over-borrow returns.** A `&self`-borrowing iterator is right; a struct where every field
  is behind a lifetime often just relocates the problem into the caller.

The asymmetry is deliberate: generous input makes the API easy to call, concrete output makes it easy
to use the result.

---

## 4. Generics or `dyn`

**Generics** for hot paths, for the common case, and where monomorphisation gives you zero-cost
abstraction. **`dyn`** when the type is genuinely chosen at runtime (plugins, a user-supplied
handler stored in a struct), when the collection is heterogeneous, or when monomorphisation bloat is
measured and real.

**Do not make everything generic.** Over-generic APIs are a top ergonomics failure:

- Call sites need turbofish because inference has nothing to work with.
- Error messages become unreadable walls of trait bounds.
- Every added bound is a breaking change waiting to happen.
- Trait objects become impossible, so the consumer cannot store your type in a `Vec`.

A good test: if a type parameter is instantiated with exactly one type in every test, doc example,
and internal use, it should not be generic. **Generic when it buys the caller something. Concrete
otherwise.**

For traits meant to be *implemented* by consumers, prefer few methods with defaults over many
required ones — every required method is work for every implementor.

---

## 5. Derive the standard traits

**`Debug` on every public type.** A public type without `Debug` is hostile: it cannot be `unwrap`ped
in a `Result`, cannot be asserted on in a test, and cannot be logged. `#[derive(Debug)]` is the
default, and a manual impl only to redact a secret (`Token(<redacted>)`).

Then, where it is meaningful: `Clone`, `Copy` (small plain data only), `Default`, `PartialEq`/`Eq`,
`Hash`, `PartialOrd`/`Ord`, `Display` for user-facing types, `Serialize`/`Deserialize` **behind a
`serde` feature** so consumers who do not need it do not compile it.

Two traps: deriving `Copy` is a permanent commitment you cannot remove without breaking callers who
relied on implicit copies; and deriving `PartialEq` on a type with a cache field or a float makes
equality mean something you did not intend. Derive on purpose, not reflexively.

---

## 6. Semver-safe evolution

Design so that the changes you *will* want are additive.

- **`#[non_exhaustive]` on every public enum and every struct likely to grow.** Adding a variant or a
  field is otherwise a major version. Apply it at design time — adding it later is itself breaking.
- **Builders over long argument lists and over a family of `new_with_*` constructors.** A builder
  absorbs new options additively, forever. `Client::builder().timeout(..).build()?` still compiles
  when three more options exist; `Client::new(url, timeout, retries)` does not.
- **Sealed traits** when you need to add methods later:
  ```rust
  mod private { pub trait Sealed {} }
  pub trait Backend: private::Sealed { /* you may add methods freely */ }
  ```
  Seal by default for traits you implement for your own types; leave open only traits you are
  inviting consumers to implement, and accept that they are then frozen.
- **Never expose a dependency's type in your public API unless you intend to version-lock with it.**
  A `pub fn client(&self) -> &reqwest::Client` means every `reqwest` major bump is a major bump for
  you and for everyone downstream. Wrap it, or re-export the dependency explicitly (`pub use reqwest;`)
  so the coupling is at least documented and the versions provably match.
- Also breaking, and routinely missed: adding a trait method without a default, tightening a bound,
  adding a public field to a non-`#[non_exhaustive]` struct, and making an optional argument
  required. `cargo semver-checks` catches many of these — see `publish-open-source`.

---

## 7. Document invariants and panics

**A function that can panic and does not say so is a bug.** Every public item that can panic gets a
`# Panics` section naming the exact condition.

```rust
/// Returns the row at `index`.
///
/// # Panics
/// Panics if `index >= self.len()`. Use [`Self::get`] for the fallible form.
///
/// # Errors
/// Returns [`Error::Closed`] if the connection has been shut down.
```

Also document: what a `None` or empty return means, ordering guarantees (or their absence), whether a
method is cancel-safe, whether a type is `Send`/`Sync`, and any complexity a caller would be
surprised by. **The invariant that only lives in your head is the one that breaks.** Prefer designing
panics away: `Result` or `Option` for anything a caller can trigger, panic only for a broken
internal invariant.

---

## 8. Before it ships

Run the surface through **`critique/api-critique`** for the review pass — this skill designs, that
one judges, and the two should not be the same reading. Then `publish-open-source` for the release
mechanics.
