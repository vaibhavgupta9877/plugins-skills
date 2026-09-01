---
name: crate-architecture
description: Structure a Rust crate or workspace — module layout, feature flags, visibility, workspace organisation, error type strategy. Do NOT use for public API surface review, that's critique/api-critique.
---

# crate-architecture

## Owns
- Module and workspace layout, feature flags, visibility policy.
- Error type architecture and internal boundaries.

## Does not own
- Public API ergonomics and naming → `api-ergonomics`.
- Publishing mechanics → `publish-open-source`.
- Async runtime design → `async-services`.
- Reviewing an existing surface → `critique/api-critique`.

**This skill assumes you know Rust.** It encodes decisions — how this codebase structures crates,
what its error policy is, what gets to be `pub` — not language reference.

---

## 1. `lib.rs` is a table of contents

**`lib.rs` declares modules, re-exports the public surface, and holds the crate docs. Nothing else.**
An implementation dump in `lib.rs` means the module boundaries were never decided.

```rust
//! What this crate is, and a working example, in the first screen.
#![warn(missing_docs)]

mod client;          // private
mod config;          // private
pub mod error;       // public because consumers match on the types

pub use client::{Client, ClientBuilder};
pub use config::Config;
```

**Consumers should never need to know an internal module path.** `mycrate::Client`, not
`mycrate::client::inner::Client`. Private modules with selective `pub use` at the root give you that,
and they let you move the implementation later without a breaking change — which is the whole point.

The corollary: `pub mod` is a commitment to that path forever. Use it only when the module is
genuinely part of the surface (`error`, sometimes `types`).

---

## 2. Visibility discipline

**`pub` is the default mistake.** Every `pub` item is a compatibility commitment you will be held to
by strangers whose code you cannot see.

- `pub(crate)` liberally — it costs nothing and keeps the surface honest.
- `pub(super)` and `pub(in path)` for genuinely local sharing.
- `pub` only for what you intend to support for the life of the major version.
- A `pub` struct with `pub` fields is the strongest commitment in the language: you can never add an
  invariant, a validation, or a computed field. Prefer private fields with accessors, or accept the
  field-struct shape deliberately for a plain data type.

**Audit the surface before every release:**

```bash
cargo public-api                        # the full surface
cargo public-api diff --deny=all        # against the last published version, in CI
```

An item that appears in that list and is not in the docs is either undocumented or should not be
public. Both are findings.

---

## 3. Workspaces — split for a reason

Split when there is a real one:

- **Separate release cadence** — the core stabilises, an extension churns.
- **An optional heavy dependency** that most consumers should not pay for.
- **A binary plus its library**, so the library is usable without the CLI's dependencies.
- **A proc-macro crate**, which *must* be its own crate — the compiler requires it.
- **An FFI/bindings crate**, so the core stays host-agnostic (see `ffi-bridge`).

**Do not split for tidiness.** Splitting creates version lockstep: every internal breaking change
becomes a coordinated multi-crate release, path dependencies must become version dependencies before
publish, and consumers now have to reason about which crate they need. Modules give you the same
organisation for free. If the crates would always be released together at the same version, they are
one crate.

When you do split, use `[workspace.dependencies]` and `[workspace.package]` so version, license,
edition, and MSRV are declared once and inherited with `.workspace = true`.

---

## 4. Error strategy — a firm policy

**`thiserror` for library error enums. `anyhow` only in binaries, tests, and build scripts.**

```rust
#[derive(Debug, thiserror::Error)]
#[non_exhaustive]
pub enum Error {
    #[error("config file {path} is not readable")]
    ConfigUnreadable { path: PathBuf, #[source] source: std::io::Error },

    #[error("server rejected the request: {status}")]
    Rejected { status: u16 },
}
```

- **`anyhow::Error` in a library's public API is a defect.** It erases the type, so the caller cannot
  match on the failure, cannot handle one case and propagate the rest, and cannot write a test that
  asserts *which* error occurred. It also drags `anyhow` into their dependency graph.
- **`#[non_exhaustive]` on every public error enum.** Without it, adding a variant is a breaking
  change — and you will add variants.
- **Preserve the `source()` chain** with `#[source]` or `#[from]`. A caller that loses the underlying
  `io::Error` has lost the only actionable information.
- **`Display` must be useful to someone who did not write the code**: what failed, and with which
  input. Lowercase, no trailing period, no "Error:" prefix — the caller adds context around it.
- One error type per crate is the default. Per-module error types are worth it only when the modules
  are genuinely independent; otherwise callers spend their time on `From` conversions.
- Panics are for bugs (a broken invariant), not for expected failure. Document every one — see
  `api-ergonomics`.

---

## 5. Feature flags are additive, always

**Enabling a feature must never remove or change behaviour.** Cargo unifies features across the whole
dependency graph: if any crate in the tree enables your `fast-math` feature, *every* consumer gets it.
A feature that changes semantics therefore breaks a consumer who never asked for it, from a distance,
with no compile error.

- No `no-foo` / `disable-foo` features. Invert them.
- **Keep `default` minimal.** Everything in `default` is what a consumer who wrote
  `--no-default-features` is choosing to lose; make that a short, principled list.
- **Document every feature** in the crate docs, in a `## Features` section, with what it enables and
  what it pulls in.
- Use `dep:` syntax and optional dependencies so a feature name is not silently also a dependency
  name: `serde = ["dep:serde", "uuid?/serde"]`.
- **Test the matrix in CI** — `--no-default-features`, `--all-features`, and each feature alone.
  `cargo hack --feature-powerset --depth 2` if the count is small enough. Feature combinations that
  fail to compile are the classic bug found by a stranger, not by you.
- `#[cfg(feature = "…")]` on public items needs `#[cfg_attr(docsrs, doc(cfg(feature = "…")))]` so
  docs.rs shows which feature gates what.

---

## 6. `no_std` is an early decision

**Decide before the first release. Retrofitting is painful** — it touches every `String`, `Vec`,
`HashMap`, `std::error::Error`, and time or I/O call in the crate.

If you support it: `#![cfg_attr(not(feature = "std"), no_std)]`, `alloc` as a separate feature where
that split is meaningful, and a **CI job that builds for a genuinely bare target** (`thumbv7em-none-eabi`)
— `cargo build --no-default-features` on a host target does not prove `no_std` compiles.

If you do not support it, say so in the README. Silence reads as "maybe", and someone will open an
issue after wiring you in.

---

## 7. Types that prevent misuse

This is where Rust earns its cost — use it, and know when to stop.

- **Newtypes over primitives** for anything with a unit or an invariant: `UserId(Uuid)`,
  `Seconds(u64)`. Two `String` parameters in a row is a call site waiting to be swapped.
- **Type-state** for protocols with an order: `Connection<Handshaking>` → `Connection<Ready>`, so a
  misuse is a compile error rather than a runtime one. Best for genuine state machines with a small
  number of states.
- **Enums over booleans** for anything with two named modes; `f(true, false)` at a call site is
  unreadable and `Mode::Strict, Trailing::Deny` is not.
- **The readability tax is real.** When a signature needs a turbofish, when the error message from a
  wrong state is three screens long, or when a contributor cannot add a method without a diagram, the
  cleverness has outrun its value. A runtime check with a clear error is sometimes the better trade —
  make it deliberately, not by accident.

---

## 8. Unsafe is quarantined

`unsafe` lives in a small, named module (`mod raw`, `mod sys`) with a safe wrapper around it, not
scattered through the crate. Every block carries a `// SAFETY:` comment stating the invariant it
relies on and why it holds here.

`#![forbid(unsafe_code)]` at the crate root when the crate genuinely has none — it is a claim
consumers look for, and CI enforces it for you. Soundness review, Miri, and the higher bar that
applies to published crates are in `perf-and-unsafe`.
