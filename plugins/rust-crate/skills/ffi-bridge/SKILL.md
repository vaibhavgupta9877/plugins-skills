---
name: ffi-bridge
description: Bridge Rust to Flutter/Dart or other hosts — flutter_rust_bridge, uniffi, cbindgen, memory ownership across the boundary, async and error mapping. Do NOT use for the Flutter-side app structure, that's flutter/flutter-architecture.
---

# ffi-bridge

## Owns
- FFI boundary design, codegen tooling choice, memory and ownership rules.
- Cross-language error and async mapping, and the multi-target build integration.

## Does not own
- The Rust core logic and crate structure → `crate-architecture`.
- How Flutter consumes the result → `flutter/flutter-architecture`.
- `unsafe` soundness review and Miri → `perf-and-unsafe`.

---

## 1. Design the boundary coarse

**Every FFI call has overhead and every exposed type is a permanent maintenance cost.** One call
returning a struct beats twelve calls returning fields — and **chatty FFI is the main performance and
complexity failure** on this boundary, not the cost of any individual call.

```rust
// WRONG — twelve crossings and twelve type mappings to render one screen
pub fn invoice_id(h: InvoiceHandle) -> String;
pub fn invoice_total(h: InvoiceHandle) -> i64;
pub fn invoice_status(h: InvoiceHandle) -> u8;

// RIGHT — one crossing, one mapped type, one screen's worth of data
pub fn load_invoice(id: String) -> Result<Invoice, AppError>;
```

- **Expose use-cases, not the object graph.** `submit_order(cart) -> OrderReceipt` is a boundary;
  `get_cart_item_count()` leaks internal structure.
- **Do not hand out handles to Rust objects that Dart then pokes at.** A handle plus twenty accessors
  is one object model spread across two languages, and it is where lifetime bugs live.
- Keep boundary types small and flat: strings, integers, byte vectors, enums, structs of those. Every
  generic, trait object, or lifetime at the boundary is either impossible or costly.
- **The boundary is an API in the sense of `api-ergonomics`** — versioning, `#[non_exhaustive]`,
  naming — that happens to have a Dart consumer.

---

## 2. Tooling

| Tool | Best for | Cost |
|---|---|---|
| **`flutter_rust_bridge`** | Flutter-only targets | Best Dart ergonomics — async, streams, and generated glue for free. You are coupled to its codegen and its version churn; a major version bump regenerates everything. |
| **`uniffi`** | Multiple host languages (Kotlin, Swift, Python, Dart) | Mozilla-maintained, conservative surface, IDL-driven. Fewer Dart niceties; Dart support is community-maintained. |
| **`cbindgen` + `dart:ffi`** | Maximum control, tiny surfaces | Maximum manual work and maximum risk. You hand-write every `extern "C"` signature, every free function, and every Dart binding. |

**Recommend `flutter_rust_bridge` when Flutter is the only host** — its async and stream support is
the difference between a natural Dart API and a callback maze, and hand-writing that glue is exactly
the code where FFI bugs live. Choose `uniffi` the moment a second host language is plausible, and raw
`cbindgen` only for a genuinely tiny surface or when you cannot take a codegen dependency.

**The generated code is a build artifact.** Regenerate it in CI and fail the build if the checked-in
output differs, or it silently drifts from the Rust source.

---

## 3. Ownership — stated as absolutes

**Every allocation that crosses the boundary has exactly one documented owner and exactly one
documented free path.** Write both in a comment on the function that returns it.

**Never free memory on the other side of the boundary from where it was allocated**, unless the API
explicitly provides a matching free function that routes back to the original allocator. Rust's
allocator and Dart's are not the same allocator, and a `free()` on the wrong side is undefined
behaviour that may work for months.

The two failure modes and why they are hard:

- **Leaks are silent.** Nothing errors; memory grows on a long-running mobile app until the OS kills
  it, and the report says "out of memory" with no attribution.
- **Use-after-free crashes at an unrelated place.** The crash is in whatever reused the memory, so
  the stack trace points at innocent code. On Android it arrives as a `SIGSEGV` in `libflutter.so`.

Practical rules:

- Prefer **copying data across the boundary** over sharing a pointer. A `Vec<u8>` copy is cheap
  relative to a lifetime bug. Only share for genuinely large buffers, and then document the contract.
- `Box::into_raw` / `Box::from_raw` in strict pairs, in the same module, with the free function
  exported next to the constructor.
- Handles are opaque integers or opaque pointers — never a pointer the host is expected to dereference.
- Test under **ASan on Android** and Valgrind on Linux; these bugs do not appear in normal testing.

---

## 4. Panics must not cross the boundary

**Unwinding into foreign code is undefined behaviour.** This is non-negotiable, not a robustness nicety.

```rust
#[no_mangle]
pub extern "C" fn do_thing(input: *const c_char) -> i32 {
    match std::panic::catch_unwind(|| { /* real work */ }) {
        Ok(Ok(())) => 0,
        Ok(Err(e)) => e.code(),
        Err(_) => { /* log it */ -1 }      // convert the panic to an error code
    }
}
```

- Wrap **every** hand-written `extern "C"` entry point. `flutter_rust_bridge` and `uniffi` do this at
  their generated boundary — verify it rather than assume it.
- `panic = "abort"` removes the unwinding problem by killing the process instead: a legitimate choice
  for a mobile app, a bad one for a plugin inside a host you do not own. Decide which you are.
- Log the panic message before converting it, or you lose the only diagnostic.

---

## 5. Errors and async

**Errors become typed results, not exceptions.** Map the Rust error enum to a Dart sealed class or a
freezed union, so the Dart side gets exhaustive `switch` rather than a stringly-typed catch.

```rust
#[derive(Debug, thiserror::Error)]
pub enum AppError {
    #[error("network unavailable")]      Network,
    #[error("not found: {0}")]           NotFound(String),
    #[error("invalid input: {field}")]   Invalid { field: String },
}
```

Keep a **stable machine-readable discriminant** on each variant. The Dart side branches on that, not
on the message — messages get improved, and translated.

**Async and isolates:**

- Rust's runtime and Dart's isolate model are separate. `flutter_rust_bridge` runs Rust work on its
  own thread pool and completes a Dart `Future` on the main isolate, so the UI thread never blocks.
- **Use a `Stream` when the Rust side produces over time** — progress, sync events, tail-follows. A
  polling loop across FFI is the usual wrong answer and burns battery.
- Cancellation does not propagate for free. If a Dart consumer can walk away from a long operation,
  pass a cancellation token in and check it — see `async-services`.
- **Dart isolates and Rust threads share no GC and no allocator.** Document which functions may be
  called from which thread; if a function is main-isolate-only, say so in the doc comment.

---

## 6. Build integration across three targets

**CI must build every target triple**, or a platform silently stops working. Use `cargo-ndk` for
Android and cross-compile the rest.

| Target | Triples | Watch for |
|---|---|---|
| Android | `aarch64-linux-android`, `armv7-linux-androideabi`, `x86_64-linux-android` (emulator) | NDK version pinned in CI; `.so` files packaged into `jniLibs/<abi>/`; 16 KB page size on new devices |
| Windows | `x86_64-pc-windows-msvc` | MSVC toolchain, not GNU — mixing them breaks linking; ship the `.dll` next to the exe |
| Linux | `x86_64-unknown-linux-gnu` | **Build on the oldest glibc you support** |

**The glibc rule is the one that bites.** glibc is forward-compatible, not backward-compatible: a
`.so` built on a current distro will not load on an older LTS, and the error is a link failure at
launch with no useful message. Build in an old container image (`ubuntu:20.04`) in CI.

Also: strip release binaries but **keep the symbols as a CI artifact**, keep the Rust and Dart
version bumps in the same commit, and check binary size — an FFI `.so` ships inside the app
(`perf-and-unsafe` covers `cargo bloat`).

---

## 7. Debugging

**A Rust panic in a stripped release binary is nearly unreadable** — you get an address, not a
frame. Configure symbol retention before you need it: `debug = 1` in the release profile, or split
symbols and archive them per version alongside the Flutter symbols (`flutter/flutter-release`).

- Wire Rust logs into the platform log (`android_logger` on Android, `tracing` to stderr on desktop)
  so `logcat` shows both sides of the boundary interleaved.
- Set `RUST_BACKTRACE=1` in debug builds; it does nothing without symbols in release.
- When a crash points into `libflutter.so` or the allocator, **suspect the boundary first** — that
  is what a use-after-free from FFI looks like.
- Reproduce on the Rust side alone with a plain `cargo test` before debugging through Dart. Most
  "FFI bugs" are ordinary Rust bugs seen through two layers of glue.
