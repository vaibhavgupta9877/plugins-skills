---
name: perf-and-unsafe
description: Profile, benchmark, and optimise Rust code, and review unsafe blocks for soundness. Use when performance is measurably a problem or unsafe is involved. Do NOT use for general code review — that's core-eng/code-review.
---

# perf-and-unsafe

## Owns
- Benchmarking, profiling, optimisation, `unsafe` soundness review, Miri.

## Does not own
- Async concurrency design and blocking-in-async → `async-services`.
- General code review → `core-eng/code-review`.
- Crate structure and where `unsafe` lives → `crate-architecture`.

---

## 1. Measure first. Always.

**Optimising un-profiled Rust is the most common waste of effort in the language.** Intuitions are
usually wrong here: the compiler already removed what you were worried about, and the real cost is in
an allocation you never noticed.

**Require a benchmark or a profile before any optimisation change.** If there is no number, the first
task is producing one — not making the code faster. Two questions before touching anything:

1. **What is the target?** "Faster" is not a target; "p99 under 10 ms" is. Without one, there is no
   point at which you stop.
2. **Is this actually hot?** A function called twice at startup does not matter, however ugly.

Then: measure → change one thing → measure again. A batch of five optimisations tells you nothing
about which one worked, and one of them is usually a regression hiding behind the others.

---

## 2. Benchmarking

**`criterion`** for anything you will make a decision on — it gives statistical confidence intervals
and detects regressions between runs, which a hand-rolled `Instant::now()` timer cannot.

```rust
fn bench(c: &mut Criterion) {
    let input = build_realistic_input();
    c.bench_function("parse", |b| b.iter(|| parse(black_box(&input))));
}
```

- **`std::hint::black_box` on inputs and results**, or LLVM deletes the work you are timing. A
  benchmark reporting picoseconds has been optimised away.
- **Benchmark in `--release`**, with setup outside the timed closure (`iter_batched` when each
  iteration needs fresh state).
- **Microbenchmarks routinely mislead**: everything is in L1, the branch predictor is trained, the
  allocator is warm, nothing contends. A 3× win that does not move the end-to-end number is not a
  win — always confirm against a realistic workload.
- Run them on a quiet machine; CI runners are too noisy for small deltas.

---

## 3. Profiling

- `perf record` + `cargo flamegraph` on Linux, or `samply` (cross-platform, opens in the Firefox
  profiler UI, usually the fastest path to an answer). `dhat-rs` for allocation profiling.
- Add `debug = true` to the release profile temporarily, or the flamegraph is all addresses.
- Read from the widest frames inward, and be suspicious of wide frames in `memcpy`, `malloc`,
  `core::fmt`, and `drop_in_place` — none are your algorithm, and all are fixable.

**What actually dominates real-world Rust hot spots**, in roughly the order you will find it:

1. **Allocation churn** — a `Vec` or `String` built and dropped in a loop.
2. **Unnecessary `.clone()`**, added to satisfy the borrow checker and never revisited.
3. **`String` where `&str` works**, and `to_string()` at every layer boundary.
4. **`Vec` reallocation** without `with_capacity` when the size is known or estimable.
5. **Formatting in hot paths** — `format!` for a key, a log line built before the level check.
6. **`HashMap` with the default SipHash** where DoS resistance is irrelevant (`rustc-hash` /
   `ahash` is often a large win for internal maps).
7. Bounds checks in a tight loop — usually removable by iterating instead of indexing, without
   `unsafe`.

**Algorithmic problems are real but rarer here** than in GC languages, because the constant factors
in idiomatic Rust are small enough that the O(n²) is visible earlier. Check the list above first.

---

## 4. Free build configuration

```toml
[profile.release]
opt-level = 3
lto = "thin"            # "fat" is sometimes better, always slower to build
codegen-units = 1       # better optimisation, slower build, no runtime cost
panic = "abort"         # binaries only — never for a library others link
strip = "symbols"       # keep a symbol artifact separately (see publish-open-source)
```

**Measure each one.** `lto = "fat"` is not always faster and is always slower to build;
`codegen-units = 1` is usually a small win and can be a large regression in build time on a big
workspace. `panic = "abort"` removes landing pads and shrinks the binary, but it breaks
`catch_unwind` — which `ffi-bridge` depends on, and which test harnesses use.

`target-cpu=native` is a real win and a real portability trap: the binary will `SIGILL` on an older
CPU. Only for a binary you build on the machine that runs it.

---

## 5. Unsafe review is a distinct, serious activity

Treat it as its own pass, not a paragraph inside a code review. The reviewer's job is to find the
input that makes the invariant false — not to check that the code looks careful.

**Every `unsafe` block has a `// SAFETY:` comment stating the invariant it relies on and why it holds
here. A block without one is a blocking finding**, regardless of whether the code is correct: an
unstated invariant is one nobody can preserve during the next refactor.

```rust
// SAFETY: `idx < self.len` is checked on the line above, and `self.ptr` is valid for
// `self.len` initialised elements for the lifetime of `&self` (established in `new`).
unsafe { &*self.ptr.add(idx) }
```

Verify, one at a time:

- **Pointer validity** — non-null, correctly aligned, pointing at initialised memory of the right
  type, and still alive. Alignment is the one people skip; a misaligned reference is UB even if the
  read appears to work on x86.
- **Aliasing** — no two live `&mut` to the same place, and no `&mut` coexisting with any `&`. This is
  the most-violated rule, and the one Miri is best at catching.
- **No data race** — the `Send`/`Sync` reasoning is written down, not assumed. A manual
  `unsafe impl Send` needs a paragraph, not a line.
- **Lifetimes not extended.** `transmute` to `'static`, or a returned reference derived from a raw
  pointer whose lifetime the signature invents, is unsound even when it happens to work.
- **No UB on panic mid-block.** If a panic can unwind while an invariant is temporarily broken —
  during a `Vec` resize, a partial move, a half-initialised buffer — the `Drop` that runs next sees
  an invalid state. Use a guard type or `ManuallyDrop`.
- **The safe wrapper is genuinely safe**: there is *no* sequence of safe calls from any consumer that
  causes UB. This is the actual bar for `unsafe` in a library, and it is much higher than "the unsafe
  block is correct".

---

## 6. Miri

```bash
cargo +nightly miri test
MIRIFLAGS="-Zmiri-strict-provenance" cargo +nightly miri test
```

**Miri catches UB that passes tests, passes review, and passes in production for a year.** Run it on
anything touching `unsafe`, and keep it as a CI job (see `publish-open-source`'s workflow).

Its limits, so you know what it does not prove: it only checks the paths your tests execute — Miri
plus 40% coverage tests 40% of your `unsafe`; it cannot run most FFI; and it is roughly 100× slower,
so the test suite may need a reduced-input mode. Pair it with `cargo fuzz` for anything parsing
untrusted input.

---

## 7. The bar for unsafe is higher in a published crate

**Unsoundness in a published crate becomes other people's crashes**, in code you cannot see, reached
by inputs you did not imagine, and reported to you as a mystery. You cannot fix it for them quickly —
a yank does not remove the version, and their lockfile still points at it.

**If safe code is within about 10% and the crate is published, prefer safe.** State the tradeoff
explicitly rather than defaulting to clever: name the measured gain, the invariant `unsafe` would
require, and who has to maintain it. If the answer is "5% faster on a path that runs once per
request", the answer is safe code.

`#![forbid(unsafe_code)]` when the crate genuinely has none — it is a claim consumers look for, and
the compiler enforces it for you. When you must use `unsafe`, isolate it per `crate-architecture`:
one small module, a safe wrapper, and `unsafe_op_in_unsafe_fn` enabled so each operation is marked.

---

## 8. Binary size

Relevant for FFI and desktop targets, where the artifact ships inside an app.

```bash
cargo bloat --release --crates          # which dependency costs what
cargo bloat --release -n 20             # the largest functions
```

`opt-level = "z"`, `lto = "fat"`, `codegen-units = 1`, `panic = "abort"`, and `strip` all shrink it.
The usual large finds are a monomorphised generic instantiated for many types, `core::fmt` pulled in
by one `format!`, and a dependency enabled with default features you do not use — check that before
reaching for build flags.
