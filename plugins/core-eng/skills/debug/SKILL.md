---
name: debug
description: Systematically diagnose a bug, crash, or unexpected behaviour by narrowing hypotheses against evidence. Use when something is broken and the cause is unknown. Do NOT use when the fix is already identified.
---

# debug

## Owns
- Root-cause analysis, hypothesis narrowing, instrumentation, reproduction.

## Does not own
- Reviewing a proposed fix → `code-review`.
- Deciding what test pins the fix → `test-strategy` (this skill names it; that one writes it).

---

## Step 0 — Read the error, all of it

Most wasted debugging time is skipped reading. Before any theory:

- Read the **full** error message and the **full** stack trace, including causes and
  suppressed exceptions. The frame you need is rarely the top one.
- Find the **first** frame in your own code, not the framework's.
- Read the logs *before* the error, not just at it. The cause is usually upstream.
- Note the exact input and state. "It fails sometimes" is not evidence; "it fails for
  orders with zero line items" is.

If you are theorising and you have not read the trace, stop and read it.

---

## Step 1 — Reproduce

**A bug that cannot be reproduced cannot be confirmed fixed.** Get to a command or a
sequence of actions that fails reliably, then shrink it: fewest steps, smallest input,
least environment.

If reproduction is genuinely impossible — production-only, timing-dependent, one customer —
say so plainly, and **instrument instead of guessing**: add logging or metrics around the
suspected path, ship it, and wait for data. Shipping a speculative fix to an
unreproduced bug means you will never know if it worked.

---

## Step 2 — Bisect the surface

Classify the bug before narrowing. Each class has a different strategy:

- **Data** — fails for some inputs, not others. Narrow by shrinking the input until it
  stops failing. Look for empty, null, unicode, very large, duplicate, boundary values.
- **Environment** — works here, fails there. Diff the environments: versions, env vars,
  OS, build flags, feature flags, data. Do not debug the code until the environments match.
- **Timing** — intermittent, or only under load. Look for shared mutable state, missing
  awaits, ordering assumptions, cache invalidation, retries. Reproduce by adding delay or
  concurrency, not by re-running and hoping.
- **Logic** — deterministic and always wrong. Bisect the code path: assert the expected
  value at the midpoint of the pipeline and see which half lies. `git bisect` when it
  used to work.

---

## Step 3 — Hypothesise, then kill it

State a **falsifiable** hypothesis and design the **cheapest** experiment that could
disprove it:

> "`total` is computed before discounts load, so discounted orders are wrong.
> Cheapest test: log `total` and `discounts.length` at line 88 for one failing order."

Rules:

- One hypothesis at a time. Two changes at once means you learn nothing from either.
- **Never change code hoping it helps.** If you cannot say what result would prove you
  wrong, you are not debugging.
- A hypothesis that survives one experiment is not confirmed — confirm it by making the
  bug appear and disappear on demand.
- Revert every instrumentation change once done, or fold the useful ones into permanent
  logging deliberately.

---

## Stack traps to check first

**Flutter** — using `BuildContext` after an `await` (async gap); `context` from a disposed
widget; `setState` after dispose; platform-channel failures that only appear on Windows or
Linux desktop; stale providers surviving a hot reload — restart before believing a hot-reload
bug; layout errors that report a symptom widget far from the offending constraint.

**Rust** — borrow and lifetime errors that are the compiler describing a design problem, not
an obstacle to route around with `clone()` or `Rc`; `unwrap` / `expect` in library code
panicking on a case the API allows; async runtime mismatch (blocking call inside an async
context, two runtimes in one process); `Drop` order; integer overflow behaving differently
in release than debug.

**Next.js** — the server/client boundary (a server-only value read in a client component,
or a `use client` file importing server code); hydration mismatch from `Date`, `Math.random`,
or `localStorage` during render; stale `fetch` / router cache making a fix look ineffective;
edge vs node runtime API differences; env vars missing at build time versus run time.

---

## Step 4 — Report the finding

When you have the cause, state all four — not just the fix:

1. **Root cause** — the specific line or interaction, and *why* it produces the symptom.
2. **Blast radius** — what else uses this path, and whether existing data is corrupted.
3. **Fix** — the change, and why this one rather than the patch at the symptom.
4. **The test that pins it** — a test that fails on today's code and passes after. Hand the
   authoring to `test-strategy`.

If the root cause implies a design problem rather than a typo, say so and hand off to
`grill-me` or `critique/architecture-critique` — do not quietly redesign a subsystem inside
a bug fix.
