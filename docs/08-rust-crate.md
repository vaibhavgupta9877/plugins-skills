# Plugin: `rust-crate`

**Enable for:** Rust projects — general-purpose crates, FFI bridges into Flutter, async backend services, and public open-source crates.
**Purpose:** Rust engineering conventions, with a strong bias toward crates that other people will actually use.
**Skills:** 6

**Framing the agent must internalise:** the model already knows Rust. None of these skills teach the borrow checker, syntax, or standard library. They encode **decisions**: how this codebase structures crates, what its error policy is, what makes a public API worth depending on, and what shipping to crates.io actually demands. Any content that reads like a Rust tutorial should be deleted.

**Public-crate posture:** the user publishes crates for general use. That raises the bar permanently — every `pub` item is a compatibility commitment, documentation is a feature not an afterthought, and semver mistakes are visible to strangers. Skills marked *(public)* apply extra rules when `publish = true`.

---

## `crate-architecture` — CORE

**description:** `Structure a Rust crate or workspace — module layout, feature flags, visibility, workspace organisation, error type strategy. Do NOT use for public API surface review, that's critique/api-critique.`

**Owns:** module and workspace layout, feature flags, visibility policy, error type architecture, internal boundaries.
**Does NOT own:** public API ergonomics → `api-ergonomics`; publishing mechanics → `publish-open-source`; async runtime design → `async-services`.

**Body must cover:**
- Module layout: `lib.rs` is a table of contents and re-export surface, not an implementation dump. Private modules with selective `pub use` at the root — consumers should never need to know the internal module path.
- **Visibility discipline (public):** `pub` is the default mistake. Use `pub(crate)` liberally; `pub` only for what you intend to support forever. Audit the public surface before every release — `cargo public-api` for diffs.
- Workspaces: split when there is a real reason (separate release cadence, optional heavy dependency, a binary plus its library, a proc-macro crate which *must* be separate). Splitting for tidiness creates version-lockstep pain. Say when not to split.
- **Error strategy, stated as a firm policy:** `thiserror` for library error enums with `#[non_exhaustive]`; `anyhow` only in binaries and tests, never in a library's public API. Every error implements `std::error::Error` and `Display` with a message useful to someone who didn't write the code. Preserve `source()` chains.
- Feature flags: additive only — enabling a feature must never remove or change behaviour, because Cargo unifies features across the dependency graph. `default` kept minimal. Document every feature in the crate docs. Test the feature matrix in CI (`--no-default-features`, `--all-features`).
- `no_std` support: decide early. Retrofitting is painful. If supported, gate `std` behind a feature and test the `no_std` build in CI.
- Type-state and newtype patterns where they prevent misuse at compile time — this is where Rust earns its cost, so use it, but note the readability tax when it gets clever.
- Unsafe: isolated in a named module with a documented invariant per block. Cross-reference `perf-and-unsafe`.

**Budget:** 190 lines.

---

## `api-ergonomics` — CORE

**description:** `Design the public API of a Rust crate for external consumers — naming, trait design, generics vs dyn, builders, semver-safe evolution. Do NOT use to review an existing API, that's critique/api-critique.`

**Owns:** designing a public surface that strangers can use correctly.
**Does NOT own:** review of existing surface → `critique/api-critique`; internal structure → `crate-architecture`.

**Body must cover:**
- Write the call site first. If the example in the README is awkward, the API is wrong — fix the API, not the example.
- Follow the Rust API Guidelines naming conventions rather than inventing local ones: `as_`/`to_`/`into_` mean specific things (borrowed→borrowed, expensive conversion, owned→owned); `iter`/`iter_mut`/`into_iter`; getters without a `get_` prefix. Consistency with the ecosystem is worth more than any personal preference.
- Accept generously, return concretely: take `impl AsRef<str>`, `impl IntoIterator`, `&[T]`; return concrete types or `impl Trait`. Do not return `Box<dyn Trait>` where a concrete type works.
- Generics vs `dyn`: generics for hot paths and zero-cost abstraction; `dyn` when the type must be chosen at runtime or when monomorphisation bloat is real. Do not make everything generic — over-generic APIs force turbofish at call sites and are a common ergonomics failure.
- Derive the standard traits where sensible (`Debug` on everything public — a public type without `Debug` is hostile; then `Clone`, `Default`, `PartialEq`, `Serialize`/`Deserialize` behind a feature).
- **Semver-safe evolution, since these are public crates:** `#[non_exhaustive]` on any enum or struct likely to grow; builders instead of long argument lists or many `new_with_*` constructors; sealed traits when you need to add methods later; never expose a dependency's type in your public API unless you intend to version-lock with it.
- Document the invariants and panic conditions of every public function. A function that can panic and doesn't say so is a bug.
- Cross-reference `critique/api-critique` for the review pass before release.

**reference/:** `guidelines-checklist.md` — condensed Rust API Guidelines checklist.

**Budget:** 190 lines.

---

## `async-services` — SUPPORTING

**description:** `Build async Rust services and backends — Tokio runtime design, concurrency, cancellation, shared state, error propagation, graceful shutdown.`

**Owns:** async architecture, runtime configuration, concurrency primitives, task lifecycle, backpressure.
**Does NOT own:** crate structure → `crate-architecture`; FFI async bridging → `ffi-bridge`; benchmarking → `perf-and-unsafe`.

**Body must cover:**
- **Library rule (public):** a library should be runtime-agnostic where possible, or gate the runtime behind a feature. A library that hard-depends on Tokio limits who can use it — state this cost before defaulting to it.
- Blocking in async is the number-one defect: file I/O, CPU work, `std::sync::Mutex` held across an `.await`. Use `spawn_blocking` or a dedicated pool. Name the symptom (stalled runtime, latency spikes on unrelated requests) so it's diagnosable.
- Shared state: `Arc<Mutex<T>>` is fine and often correct — do not reach for channels reflexively. Use `tokio::sync::Mutex` only when the guard crosses an `.await`; otherwise `std::sync::Mutex` is faster. `RwLock` only when reads genuinely dominate.
- Cancellation safety: `select!` can drop a future mid-operation. Any future used in `select!` must be cancel-safe or wrapped. This causes silent data loss and is hard to debug — call it out explicitly.
- Structured concurrency: `JoinSet` over detached `spawn`, so failures surface. A detached task that panics disappears silently.
- Backpressure: bounded channels by default. Unbounded channels convert a throughput problem into an out-of-memory crash.
- Graceful shutdown: signal handling, cancellation token propagation, draining in-flight work with a timeout, closing pools. Most services skip this and lose requests on every deploy.
- Error propagation across task boundaries, and `tracing` for structured spans — logging without spans is unusable in concurrent code.
- Timeouts on every external call. No exceptions.

**Budget:** 150 lines.

---

## `ffi-bridge` — SUPPORTING

**description:** `Bridge Rust to Flutter/Dart or other hosts — flutter_rust_bridge, uniffi, cbindgen, memory ownership across the boundary, async and error mapping.`

**Owns:** FFI boundary design, codegen tooling, memory/ownership rules, cross-language error and async mapping, build integration.
**Does NOT own:** the Rust core logic → `crate-architecture`; the Flutter side consumption → `flutter/flutter-architecture`.

**Body must cover:**
- **Design the boundary to be coarse.** Every FFI call has overhead and every exposed type is a maintenance cost. One call returning a struct beats twelve calls returning fields. Chatty FFI is the main performance and complexity failure.
- Tooling choice with tradeoffs stated: `flutter_rust_bridge` (best Dart ergonomics, async and stream support, generated glue, but couples you to its codegen and version churn); `uniffi` (multi-language, Mozilla-maintained, more conservative surface); raw `cbindgen` + `dart:ffi` (maximum control, maximum manual work and maximum risk). Recommend `flutter_rust_bridge` for Flutter-only targets and say why.
- **Ownership rules, stated as absolutes:** every allocation crossing the boundary has exactly one documented owner and one documented free path. Never free memory on the other side of the boundary from where it was allocated unless the API explicitly provides a matching free function. Leaks here are silent; use-after-free here crashes in a way that's very hard to attribute.
- **Panics must not cross the FFI boundary** — unwinding into foreign code is undefined behaviour. Catch at the boundary (`catch_unwind`) and convert to an error result. This is non-negotiable.
- Errors become typed results, not exceptions. Map to a Dart-friendly sealed representation.
- Async: how the Rust runtime relates to Dart's isolate model, and where a stream is the right shape versus a future.
- Threading: Dart isolates and Rust threads have no shared GC — document what may be called from which thread.
- Build integration across all three targets: Android NDK cross-compilation and JNI packaging, Windows MSVC toolchain, Linux glibc version constraints (build on the oldest supported glibc or the binary won't run on older distros). CI must build all target triples.
- Debugging: symbols on both sides, and the fact that a Rust panic in release with a stripped binary is nearly unreadable — configure symbol retention.

**Budget:** 170 lines.

---

## `publish-open-source` — CORE

**description:** `Prepare and publish a Rust crate to crates.io — docs, README, licensing, semver, MSRV, CI, changelog, release process. Use before any publish. Do NOT use for the general prod readiness gate, that's secure-ship/prod-readiness-gate.`

**Owns:** the publish pipeline and everything an external consumer judges the crate by.
**Does NOT own:** go/no-go gating → `secure-ship/prod-readiness-gate` (that skill *calls* this one's checklist); API design → `api-ergonomics`.

**Body must cover:**
- **Documentation is the product.** For a public crate, docs.rs is where adoption is won or lost. Require: `#![warn(missing_docs)]` (ideally `deny`), a crate-level `//!` doc that explains what this is and shows a working example in the first screen, doc examples on every significant public item, and `#![doc = include_str!("../README.md")]` so README and crate docs cannot diverge.
- Doctests are tests. They must compile and pass in CI — this is what prevents the classic "README example doesn't work" failure.
- `Cargo.toml` completeness: `description`, `license` (`MIT OR Apache-2.0` as the ecosystem default), `repository`, `documentation`, `keywords`, `categories`, `readme`, `rust-version` (MSRV), `exclude` to keep the package small.
- **MSRV:** declare it, test it in CI with a pinned toolchain job, and treat raising it as a minor-version event with a changelog note.
- **Semver, taken seriously:** breaking changes are not just signature changes — adding a variant to a non-`#[non_exhaustive]` enum, adding a trait method without a default, adding a public field, changing a re-exported dependency's major version, and tightening a bound are all breaking. Run `cargo semver-checks` before publish. Cross-reference `critique/api-critique` for the human review pass.
- `CHANGELOG.md` in Keep a Changelog format, written for consumers, updated in the same PR as the change — not reconstructed from git log at release time.
- CI matrix: stable, beta, MSRV; `--no-default-features` and `--all-features`; `clippy -D warnings`; `cargo fmt --check`; `cargo deny`; docs build with `--cfg docsrs`.
- Release sequence: version bump → changelog → `cargo publish --dry-run` → verify the packaged file list contains nothing unintended → tag → publish → verify docs.rs built successfully (docs.rs builds fail after publish and cannot be retried by re-publishing the same version).
- **Yanking is not deletion.** A published version is public forever. Check for secrets in the package before publishing, not after.
- Contribution surface: README with install/usage/MSRV/licence, `CONTRIBUTING.md`, issue templates, and a stated maintenance intent. Say honestly if you're not going to maintain it — abandoned crates with confident READMEs waste other people's time.

**reference/:** `release-checklist.md`, `ci-workflow.md`.

**Budget:** 200 lines.

---

## `perf-and-unsafe` — SUPPORTING

**description:** `Profile, benchmark, and optimise Rust code, and review unsafe blocks for soundness. Use when performance is measurably a problem or unsafe is involved.`

**Owns:** benchmarking, profiling, optimisation, `unsafe` soundness review, Miri.
**Does NOT own:** async concurrency design → `async-services`; general code review → `core-eng/code-review`.

**Body must cover:**
- **Measure first, always.** Optimising un-profiled Rust is the most common waste of effort in the language, because intuitions about what's slow are usually wrong. Require a benchmark or profile before any optimisation change.
- Benchmarking: `criterion` for statistical rigour, `std::hint::black_box` to defeat the optimiser, benchmark in `--release`, and be aware that microbenchmarks routinely mislead about real workloads.
- Profiling: `perf` + flamegraph on Linux, `cargo-flamegraph`, `samply`. Look for allocation churn, unnecessary `clone`, `String` where `&str` works, `Vec` reallocation without `with_capacity`, and formatting in hot paths — these dominate real-world Rust hot spots far more often than algorithmic issues.
- Build config that's free: `opt-level`, `lto = "thin"` or `"fat"`, `codegen-units = 1`, `panic = "abort"` for binaries. Measure each — LTO is not always faster and always slower to build.
- **Unsafe review, treated as a distinct and serious activity:** every `unsafe` block has a `// SAFETY:` comment stating the invariant it relies on and why it holds. A block without one is a blocking finding. Verify: pointer validity and alignment, no aliasing violation (`&mut` uniqueness), no data race, lifetimes not extended, no UB on panic mid-block.
- Run `cargo miri test` on anything touching unsafe. Miri catches UB that passes tests and passes review.
- **The bar for unsafe in a public crate is higher**, because unsoundness becomes other people's crashes. If safe code is within ~10% and the crate is published, prefer safe. State this tradeoff explicitly rather than defaulting to clever.
- `cargo bloat` and binary size for the FFI/desktop targets, where shipped size actually matters.

**Budget:** 150 lines.
