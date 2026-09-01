---
name: async-services
description: Build async Rust services and backends — Tokio runtime design, concurrency, cancellation, shared state, error propagation, graceful shutdown. Do NOT use for crate structure, that's crate-architecture.
---

# async-services

## Owns
- Async architecture, runtime configuration, concurrency primitives, task lifecycle, backpressure.

## Does not own
- Crate and module structure → `crate-architecture`.
- Bridging async across an FFI boundary → `ffi-bridge`.
- Benchmarking and profiling → `perf-and-unsafe`.

---

## 1. Libraries should not choose the runtime

**A library that hard-depends on Tokio limits who can use it.** Someone on `async-std`, `smol`, or an
embedded executor cannot use your crate at all, and they will not open an issue — they will pick a
different crate.

- Depend on **traits, not the runtime**: `std::future::Future`, `futures::Stream`,
  `AsyncRead`/`AsyncWrite`. Gate the runtime behind a feature
  (`tokio = { version = "1", optional = true }`).
- If you genuinely need `tokio::time` or `tokio::spawn` in the core, **say so in the README** — a
  runtime dependency discovered at integration time is worse than one stated up front.
- **Binaries choose freely.** Tokio multi-thread is the right default for a service, with the worker
  count left alone unless measured, and `current_thread` for tests and single-core targets.

---

## 2. Blocking in async is the number-one defect

Async tasks are cooperatively scheduled. A task that does not yield holds its worker thread, and
**every other task on that thread stalls** — including ones with nothing to do with the blocked work.

What blocks, and gets missed: `std::fs` I/O; CPU work including **password hashing, which is
deliberately slow**; compression and image processing; `std::sync::Mutex` held across an `.await`;
any synchronous database or HTTP client; `std::thread::sleep` and blocking channel `recv()`.

**Symptom to recognise:** latency spikes on requests that share nothing with the slow one, tail
latency far worse than mean, throughput collapsing at a specific concurrency. Idle CPU plus rising
latency means a blocked worker — look there before anything else.

The fix: `spawn_blocking` for occasional blocking work, a dedicated `rayon` pool for sustained CPU
work. `spawn_blocking`'s pool is bounded and is for *waiting*, not for saturating cores.

---

## 3. Shared state — `Arc<Mutex<T>>` is often correct

**Do not reach for channels reflexively.** A shared counter, cache, or config behind `Arc<Mutex<_>>`
is simple and fast. Channels hand off *work*; an actor wrapped around a `HashMap` is usually more
code and more latency for the same result. The choice that actually matters:

- **`std::sync::Mutex`** when the guard does **not** cross an `.await` — faster, and the right
  default. Clippy's `await_holding_lock` catches the mistake.
- **`tokio::sync::Mutex`** only when the guard must be held across an `.await`. Needing it usually
  means the critical section is too large — do the I/O outside the lock.
- **`RwLock`** only when reads genuinely dominate and the critical section is non-trivial; under
  mixed load it is often slower than a `Mutex` and can starve writers.
- **`Arc<T>` alone** when `T` is immutable after construction — the cheapest option, and often
  reachable with a small restructure. `dashmap` or sharding only once a lock is a *measured*
  bottleneck (`perf-and-unsafe`).

---

## 4. Cancellation safety

**`select!` drops the losing futures mid-operation.** If a dropped future had already consumed a
message, taken bytes off a socket, or partially advanced a state machine, that work is silently lost.

```rust
// WRONG — a new process() future each iteration; partial work vanishes when shutdown wins
loop {
    tokio::select! {
        _ = shutdown.cancelled() => break,
        res = process(&mut stream) => { … }
    }
}

// RIGHT — pin it outside the loop so the future survives an iteration
let fut = process(&mut stream);
tokio::pin!(fut);
loop {
    tokio::select! {
        _ = shutdown.cancelled() => break,
        res = &mut fut => { … }
    }
}
```

**This causes silent data loss and is very hard to debug** — nothing errors, the message is simply
gone, and only under the timing that triggers the other branch.

Check each future's docs for a "Cancel safety" section (`Receiver::recv` is cancel-safe;
`read_exact` is not). If it is not cancel-safe, pin it outside the loop or move it into a task with
its own completion signal. **Document the cancel safety of async functions you publish**
(`api-ergonomics`).

---

## 5. Structured concurrency

**`JoinSet` over detached `spawn`.** A detached task that panics or errors disappears silently: no
log, no failed request, no exit code. You find out when the data is missing.

```rust
let mut set = JoinSet::new();
for item in batch { set.spawn(handle(item)); }
while let Some(res) = set.join_next().await {
    match res {
        Ok(Ok(())) => {}
        Ok(Err(e)) => tracing::error!(error = %e, "item failed"),
        Err(e) => tracing::error!("task panicked or was cancelled: {e}"),
    }
}
```

`JoinSet` also cancels remaining tasks on drop, so a failing scope leaves no orphans. Use
`try_join!` for a fixed set of heterogeneous futures and `buffer_unordered(n)` for bounded fan-out —
**never `join_all` over an unbounded collection**, which starts everything at once. Detached `spawn`
is legitimate for real background services (a flusher, a reaper); give those an explicit handle and a
cancellation token so shutdown can reach them.

---

## 6. Backpressure

**Bounded channels by default.** An unbounded channel converts a throughput problem into an
out-of-memory crash: the producer never slows down, the queue grows, and the process is killed —
usually in production, usually at the worst moment, and with no useful error.

- `mpsc::channel(n)` with a deliberate `n` — the capacity states how much lag you absorb before
  pushing back. Write down the reasoning.
- On a full channel: `send().await` (backpressure) or `try_send` and shed load with a clear error.
  **Decide which**; silently blocking a request handler forever is not a third option.
- `Semaphore` to bound expensive concurrent operations, and rate-limit outbound third-party calls
  before they rate-limit you.
- `unbounded_channel` is defensible only when the producer is provably bounded (a fixed set of
  shutdown signals). Justify it in a comment.

---

## 7. Graceful shutdown

**Most services skip this and lose in-flight requests on every deploy** — a small constant background
error rate nobody attributes to deployments.

1. Catch `SIGTERM` and `SIGINT` (`tokio::signal`).
2. Stop accepting new work — close the listener, stop consuming from the queue.
3. Broadcast cancellation via a `CancellationToken` propagated into every task.
4. **Drain in-flight work with a timeout.** If the deadline passes, log what is still running and
   exit anyway — a hung shutdown gets SIGKILLed, which is worse. Keep the drain timeout shorter than
   the orchestrator's kill grace period, or this step never completes.
5. Flush: close pools, flush tracing and metrics exporters, commit offsets. An unflushed exporter
   means the logs explaining the shutdown never arrive.

---

## 8. Errors and tracing across tasks

- A task boundary is where errors go to die. Every spawned task's `Result` is joined and handled, or
  logged at the point of failure — never dropped.
- **`tracing`, not `log`.** Interleaved concurrent log lines with no correlation are unusable;
  logging without spans in async code is close to no logging at all.
- `#[instrument]` on handlers with the identifying fields (`request_id`, `user_id`) on the span, so
  every event inherits them. Use `.instrument(span)` when spawning or the task loses its parent.
- A panicking task aborts only that task by default. Log via a panic hook, and consider
  `panic = "abort"` for a service so a poisoned process dies rather than limping.

---

## 9. Timeouts on every external call. No exceptions.

Every network call and database query gets a `tokio::time::timeout`. **A missing timeout is not a
slow request — it is a permanently leaked task**, and a slow dependency becomes an outage as tasks
pile up until memory or the connection pool is gone.

Set it from the caller's budget, not from a guess at the dependency's speed, and pair every timeout
with a retry policy that has a cap and jittered backoff. Uncapped retries turn one slow dependency
into a self-inflicted denial of service.
