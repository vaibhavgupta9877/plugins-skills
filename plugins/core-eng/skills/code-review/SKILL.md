---
name: code-review
description: Review a diff, PR, or changeset for correctness, regressions, and structural problems. Do NOT use to audit a whole system's design — that's critique/architecture-critique.
---

# code-review

## Owns
- Diff-time review: correctness, edge cases, regression risk.
- Review comment authoring, with severity and a concrete fix.
- Calling out what the diff should have included and didn't.

## Does not own
- Holistic architecture audits with no diff → `critique/architecture-critique`.
- Security threat modelling → `secure-ship/threat-model`.
- Public API shape and ergonomics → `critique/api-critique`.
- Release gating → `secure-ship/prod-readiness-gate`.

---

## Step 1 — Read enough to judge

Read the diff **and enough surrounding code to judge it**. A diff reviewed in isolation
misses exactly the bugs that matter: the caller that passes null, the second call site that
was not updated, the invariant enforced three functions up.

For every changed function, open the callers. For every changed type, find its other uses.
For every removed line, ask what depended on it.

Get the diff with `git diff <base>...HEAD` (or `gh pr diff <n>`). Read the PR description
for intent, then judge the code against the intent — not against the description's claims.

---

## Step 2 — Priority order

Work down this list. **Never lead with style.**

1. **Correctness** — does it do what it claims, for all inputs it will actually receive?
2. **Data loss / security regression** — destructive writes, missing auth check, widened
   permission, secret in the diff, injection at a trust boundary.
3. **Error handling** — what happens on the failure path, and does the user find out?
4. **Concurrency** — shared mutable state, races, ordering assumptions, missing locks or
   transactions.
5. **Performance** — only where it will actually bite: N+1, unbounded query, work in a loop.
6. **Structure** — is this the right place for this code, and will the next person find it?
7. **Style** — last, and only if it violates the repo's own convention.

---

## Step 3 — Explicit checks

Run this list against every diff:

- Unhandled error paths; `catch` blocks that swallow silently or log-and-continue.
- `null` / `None` / `?` / `unwrap` / `!` misuse — every force-unwrap is a claimed invariant;
  is it actually guaranteed?
- Off-by-one in indices, ranges, pagination, and retry counts.
- Unbounded loops, unbounded queries, missing `LIMIT`, missing timeout.
- N+1 queries — a DB or network call inside a loop over user-controlled data.
- Missing transactions where two writes must both land, or neither.
- Missing input validation at trust boundaries — HTTP handler, CLI arg, deserialised
  payload, IPC/platform-channel message.
- Mutation of shared state: globals, singletons, captured references, module-level caches.
- Resource lifecycle: files, sockets, subscriptions, controllers, streams — opened and
  never closed / disposed / cancelled.
- Migrations that are not backwards compatible with the currently deployed code.
- Feature flags with no off-path, and dead code left behind a permanently-on flag.

---

## Step 4 — Write the comments

Every comment states **severity** and **the concrete fix**. A comment without a proposed
fix is noise the author has to translate.

- **blocking** — merging this causes a bug, data loss, or a security regression.
- **should-fix** — real problem, does not have to block the merge; say what happens if ignored.
- **nit** — style or preference. Label it honestly, and cap it: more than two or three nits
  in a review means you have stopped reviewing and started editing.

Format:

```
[blocking] src/api/orders.ts:88
`total` is computed before the discount rows are loaded, so any order with a
discount is charged the pre-discount amount.
Fix: move the computation below the `await loadDiscounts(order)` on line 94, or
compute it inside `buildInvoice` where both are in scope.
```

Anchor every comment to `file:line`. State the failure — the input or state that produces
the wrong output — not just the smell.

---

## Step 5 — What the diff should have included

Call out omissions explicitly. These are findings, not afterthoughts:

- **Tests** — is there a test that fails without this change? For a bug fix this is
  mandatory (see `test-strategy`).
- **Migration** — schema change without a migration, or a migration without a rollback.
- **Docs / changelog** — public API or user-visible behaviour changed.
- **Config / env** — a new required env var with no default and no documentation.
- **Telemetry** — a new failure mode with no way to observe it in production.

---

## Step 6 — Verdict

Close with one of:

- **Approve** — "this looks correct" when it does. Say it plainly.
- **Approve with should-fixes** — list them; merging is the author's call.
- **Request changes** — only when there is at least one blocking finding.

**Manufactured findings train the user to ignore reviews.** If a clean diff is clean, say
so and stop. Padding a review with invented concerns to look thorough is the single fastest
way to make future reviews worthless.
