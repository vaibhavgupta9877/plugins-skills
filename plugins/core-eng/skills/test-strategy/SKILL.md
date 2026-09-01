---
name: test-strategy
description: Decide what to test at which layer, and write the tests. Covers unit, integration, and end-to-end across Flutter, Rust, and Next.js. Do NOT use for manual QA checklists or prod readiness — that's secure-ship/prod-readiness-gate.
---

# test-strategy

## Owns
- Test layer selection; test authoring; fixtures and test data.
- Coverage judgement; e2e flow selection; flake elimination.

## Does not own
- Release gating and manual QA checklists → `secure-ship/prod-readiness-gate`.
- Performance benchmarking → `rust-crate/perf-and-unsafe`, `web-ui/web-performance`.
- Reviewing someone else's tests in a diff → `code-review`.

---

## The layer decision rule

- **Unit** — pure logic, branching, edge cases, error mapping, parsing, calculations.
  Fast, no I/O. This is where the *number* of tests lives.
- **Integration** — anything crossing a boundary: database, filesystem, network, platform
  channel, IPC, a real serializer. One test per boundary contract, not per method.
- **End-to-end** — **only** the 3–7 flows whose breakage means the product is down.
  Sign-in. Checkout. The one thing the app is for.

**Do not e2e-test what a unit test can catch.** An e2e test that verifies a validation
error message is a slow, flaky unit test wearing a costume.

Pick the layer by asking: *what would have to be wrong for this test to fail?* If the answer
is "one function", it is a unit test. If it is "the wiring between two systems", integration.
If it is "the product", e2e.

---

## Per-stack tooling

**Flutter** — `flutter_test` for units and widgets; `integration_test` for real-device
flows; golden tests for UI regressions; `mocktail` for doubles. Widget tests are cheap and
catch layout and state bugs; use them liberally. Platform-channel code needs an integration
test on the real target (Android / Windows / Linux), not a mock.

**Rust** — `#[cfg(test)]` modules for units, `tests/` for integration against the public
API only, `proptest` for invariants ("round-trip never loses data", "sort output is
ordered"), doctests as executable public-API contracts. If a doctest is hard to write, the
API is hard to use — that is a finding for `critique/api-critique`.

**Next.js / T3** — Vitest for units, React Testing Library for components (query by role
and text, never by class), Playwright for e2e, direct tRPC procedure calls for API-layer
tests. Test server and client boundaries separately; a component test that needs the whole
app rendered is testing the wrong thing.

See `reference/stack-recipes.md` for setup, config, and directory layout per stack.

---

## Rules

**Coverage is a diagnostic, not a target.** Use it to find untested files you forgot, never
as a number to hit. Chasing a percentage produces tests that call code and assert nothing.

**Every bug fix ships with a test that fails without the fix.** Write the test first,
confirm it fails for the right reason, then fix. A regression test that passes on the
unfixed code is testing nothing.

**Assert on behaviour, not implementation.** A test that breaks when you rename a private
method is a maintenance tax with no safety value.

**Golden / snapshot tests** — valuable in Flutter, where a golden pins real rendered UI and
catches genuine visual regressions. Dangerous on the web, where snapshots churn on every
markup change and get blanket-updated without being read. In web code, snapshot only stable
serialised data, never rendered markup.

**No sleeps.** Wait on a condition — an element state, a response, a stream event. Static
timeouts are the primary source of flakes and they get longer forever.

**Tests own their data.** Each test creates what it needs and cleans up, or runs in a
transaction that rolls back. Shared mutable fixtures produce order-dependent suites that
fail only in CI.

**A flaky test is a failing test.** Quarantine or fix it the day it appears. Hunt flakes by
repeating the suite (`--repeat-each` in Playwright, `cargo test -- --test-threads` variation,
re-runs in CI). A suite people re-run until green protects nothing.

---

## E2E specifics

- **Selectors**: user-facing locators first — `getByRole`, `getByText`, `getByLabel`. Then
  explicit `data-testid`. Never CSS descendant chains like `div > div > .btn-blue`.
- **Configure for failure diagnosis**: trace on first retry, video and screenshot on
  failure. A failed CI run with no artifact costs an hour.
- **Isolation**: limit parallel workers when tests share a database, or give each worker
  its own schema. Prefer the second.
- **API + UI assertions together**: after the UI says "saved", assert the record actually
  exists via the API or DB. Half of e2e value is catching the lie.
- **Environment**: base URL and credentials from env vars, never hardcoded. No real secrets
  in the repo.
- Use `npx playwright codegen` to draft selectors and `--debug` / trace viewer to diagnose —
  then rewrite the generated code by hand. Generated tests are a starting point, not output.

---

## Workflow when asked to "add tests"

1. **Scope it.** Which module, and at which layer. If the ask is "test the app", derive the
   critical flows first and confirm them with the user — that is 5 e2e tests, not 500.
2. **Map what exists.** Read the current test setup and match its conventions; do not
   introduce a second test runner.
3. **List the scenarios before writing any.** Group by layer, and show the list. Cheap to
   correct on a list, expensive after they are written.
4. **Write the highest-value layer first** — usually units on the logic that is actually
   hard, not e2e on the happy path.
5. **Prove they fail.** Break the code and confirm each test catches it. A test that has
   never failed has never been verified.
