# Rust API Guidelines — condensed checklist

Distilled from the Rust API Guidelines (rust-lang/api-guidelines), reordered by how often each item
actually bites, and annotated with the failure it causes. Read it as a pre-publish pass, not as a
tutorial.

---

## Naming

- [ ] Casing follows RFC 430 — `UpperCamelCase` types and traits, `snake_case` functions and modules,
      `SCREAMING_SNAKE_CASE` consts and statics.
- [ ] Conversions use `as_` (free, borrowed→borrowed), `to_` (expensive), `into_` (consuming).
      *A `to_` that is free, or an `as_` that allocates, misleads about cost.*
- [ ] Getters have **no** `get_` prefix. `get_` only for the fallible-index sense (`get`, `get_mut`,
      `get_unchecked`).
- [ ] Iterator methods are `iter`, `iter_mut`, `into_iter`, returning `Iter`, `IterMut`, `IntoIter`.
- [ ] Iterator type names match the method that produces them.
- [ ] Feature names do not contain `use-`, `with-`, or `-support` — just the thing: `serde`, `std`.
- [ ] Names use consistent word order across the crate (`RowError`/`ColError`, not
      `RowError`/`ErrorCol`).
- [ ] A type's name does not stutter with its module: `error::Error`, not `error::CrateError`.

---

## Interoperability — the traits people expect to exist

- [ ] **`Debug` on every public type.** *Without it, `Result::unwrap`, `assert_eq!`, and `dbg!` all
      fail to compile for your users.*
- [ ] Eagerly implement where meaningful: `Clone`, `Copy` (small POD only), `Default`, `PartialEq`,
      `Eq`, `Hash`, `PartialOrd`, `Ord`.
- [ ] `Display` for types a user will see; `Display` output does **not** duplicate `Debug`.
- [ ] Error types implement `std::error::Error` and return a real `source()` chain.
- [ ] `serde::Serialize`/`Deserialize` **behind a feature**, never unconditional.
- [ ] `From`/`TryFrom`/`AsRef`/`FromStr` where the conversion is obvious; never `From` for a lossy
      or fallible conversion (that is `TryFrom`).
- [ ] Collections implement `FromIterator` and `Extend`.
- [ ] Types are `Send` + `Sync` where possible. *A non-`Send` type silently disqualifies your crate
      from most async code; if it cannot be `Send`, document why.*
- [ ] Public types are `'static` where possible — a lifetime parameter on a public type propagates
      into every consumer's struct.

---

## Macros

- [ ] `macro_rules!` macros are exported with `#[macro_export]` and documented like any public item.
- [ ] Derive macros only generate trait impls, nothing else.
- [ ] Item macros work in any position an item works, and compose in a `macro_rules!` body.
- [ ] Macro errors point at the user's code, not at macro internals.

---

## Documentation

- [ ] Every public item is documented. `#![warn(missing_docs)]`, ideally `deny`.
- [ ] Crate-level `//!` docs explain what the crate is **and show a working example in the first
      screen**. This is the single highest-leverage paragraph in the crate.
- [ ] Examples use `?`, not `unwrap`, so they model correct code.
- [ ] `# Errors` on every `Result`-returning function, naming which variants and when.
- [ ] `# Panics` on every function that can panic, naming the exact condition.
- [ ] `# Safety` on every `unsafe fn`, stating the caller's obligations.
- [ ] Prose links to related items with intra-doc links (`[`Self::get`]`, `[`crate::error::Error`]`).
- [ ] `Cargo.toml` has `description`, `license`, `repository`, `documentation`, `keywords`,
      `categories`, `readme`, `rust-version`.
- [ ] Release notes exist and are consumer-facing (`CHANGELOG.md`), not a git log.
- [ ] Docs show which features gate which items: `#[cfg_attr(docsrs, doc(cfg(feature = "…")))]`.

---

## Predictability

- [ ] Smart pointers do not add inherent methods — they would shadow the target's methods through
      `Deref`.
- [ ] Conversions live on the most specific type involved.
- [ ] Functions with a clear receiver are methods, not free functions.
- [ ] Functions do not take out-parameters — return a tuple or a struct.
- [ ] Operator overloads are unsurprising. `Add` that mutates is a trap.
- [ ] `Deref` and `DerefMut` only for genuine smart pointers, never as inheritance.
- [ ] Constructors are `new` (or a builder) — not `create`, `make`, or `from_config`.

---

## Flexibility

- [ ] Functions expose intermediate results the caller would otherwise recompute
      (`HashMap::insert` returning the old value is the model).
- [ ] Callers control where data is allocated — take `&mut Vec` to fill rather than always returning
      a fresh one, where it matters.
- [ ] Arguments are generic over ownership (`impl AsRef<str>`, `impl Into<String>`) where both forms
      are plausible — **without** making the whole type generic.
- [ ] Traits are object-safe if being used as a trait object is plausible. *Adding a generic method
      later silently removes object safety and is breaking.*

---

## Type safety

- [ ] Newtypes for units and invariants. *Two adjacent `u64` parameters will be transposed.*
- [ ] Arguments convey meaning through types — an enum, not a `bool`.
- [ ] `Option<T>` for genuinely optional values, never a sentinel (`-1`, `""`, `0`).
- [ ] `#[must_use]` on anything whose result being ignored is a bug — builders, `Result`-like types,
      pure transforms.
- [ ] Builders are used for anything with more than three or four options.

---

## Dependability

- [ ] Functions validate arguments, and the choice between panic and `Result` is deliberate and
      documented.
- [ ] Destructors never fail, and never block indefinitely. *A blocking `Drop` deadlocks async code.*
- [ ] The crate does not leak on the normal path; `mem::forget` is not part of the design.

---

## Future proofing (the semver surface)

- [ ] **`#[non_exhaustive]` on every public enum**, and on structs likely to gain fields.
- [ ] Sealed traits for anything not intended to be implemented downstream.
- [ ] Structs have private fields unless they are genuinely plain data.
- [ ] Newtypes wrap types you do not want to be part of your contract.
- [ ] **No dependency type appears in the public API** unless you accept version-locking to it — or
      you `pub use` it explicitly so the coupling is documented and versions provably match.
- [ ] MSRV is declared (`rust-version`) and tested in CI.

---

## Necessities

- [ ] Public dependencies of a stable crate are themselves stable (≥ 1.0).
- [ ] The crate and its dependencies have a permissive licence (`MIT OR Apache-2.0` is the ecosystem
      default), with both licence files present.
- [ ] `cargo deny check` passes — licences, advisories, duplicate versions, banned crates.

---

## Verification commands

```bash
cargo public-api                              # what is actually public
cargo public-api diff --deny=all              # vs. the last published version
cargo semver-checks check-release             # mechanical semver violations
cargo doc --no-deps --all-features            # then read it as a stranger would
cargo clippy --all-targets --all-features -- -D warnings
cargo deny check
cargo +nightly udeps                          # dependencies you no longer use
```

**None of these replace reading the docs as a new user.** Open the rendered docs.rs page, start at
the crate root, and try to accomplish the crate's main task using only what is on the page. Every
time you have to look at the source, that is a finding — hand them to `critique/api-critique`.
