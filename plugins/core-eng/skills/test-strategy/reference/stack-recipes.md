# Stack recipes — setup, config, layout

Detect the existing setup first. These are defaults for a repo with nothing, not
a migration target for a repo that already has tests.

---

## Flutter

**Deps** (`dev_dependencies`): `flutter_test`, `integration_test` (sdk), `mocktail`,
`golden_toolkit` (optional, for device-size goldens).

**Layout**
```
test/
  unit/            # pure dart logic, no widgets
  widget/          # single-widget and screen tests
  golden/          # *.png baselines + tests
integration_test/
  app_test.dart    # real-device flows
```

**Commands**
```
flutter test                          # unit + widget
flutter test --update-goldens         # regenerate baselines — review the diff
flutter test integration_test         # on a connected device/emulator
```

**Notes**
- Goldens are platform-sensitive. Pin the font loader and generate on one platform
  (usually CI Linux) or they churn per developer.
- Riverpod: override providers in `ProviderScope(overrides: [...])` rather than
  mocking deep dependencies.
- Platform channels: mock with `TestDefaultBinaryMessengerBinding` for widget tests,
  but keep one real integration test per channel on the actual target OS.

---

## Rust

**Layout**
```
src/lib.rs         #[cfg(test)] mod tests — unit, sees private items
tests/             integration — public API only, one binary per file
benches/           criterion, if perf matters
```

**Deps** (`[dev-dependencies]`): `proptest` or `quickcheck`, `insta` for snapshots,
`tokio` with `macros`+`rt` for async tests, `assert_cmd` + `predicates` for CLIs.

**Commands**
```
cargo test                    # units + integration + doctests
cargo test --doc              # public-API contracts only
cargo test -- --nocapture     # see println from tests
```

**Notes**
- Doctests are documentation that cannot rot. Every public item worth using gets one.
- `proptest` earns its keep on invariants: round-trips, ordering, idempotence.
- Async: match the runtime the crate actually uses; a `#[tokio::test]` in a crate that
  ships `async-std` proves nothing.
- Never `unwrap()` in library code just to satisfy a test — that is the test finding a bug.

---

## Next.js / T3

**Deps**: `vitest`, `@vitejs/plugin-react`, `@testing-library/react`,
`@testing-library/user-event`, `jsdom`, `@playwright/test`.

**Layout**
```
src/**/*.test.ts        # colocated units
src/**/*.test.tsx       # component tests
e2e/                    # playwright specs
  fixtures/             # auth state, seeded data
playwright.config.ts
vitest.config.ts
```

**vitest.config.ts essentials**
```ts
test: { environment: 'jsdom', setupFiles: ['./vitest.setup.ts'], globals: true }
```

**playwright.config.ts essentials**
```ts
use: { baseURL: process.env.BASE_URL ?? 'http://localhost:3000',
       trace: 'on-first-retry', video: 'retain-on-failure', screenshot: 'only-on-failure' },
webServer: { command: 'pnpm build && pnpm start', url: 'http://localhost:3000',
             reuseExistingServer: !process.env.CI },
workers: process.env.CI ? 2 : undefined,
```

**Notes**
- tRPC: test procedures by calling the router directly with a built context. Faster and
  more precise than going through HTTP.
- Auth in e2e: sign in once in a setup project, save `storageState`, reuse it. Do not log
  in through the UI in every test.
- Prisma: run e2e against a dedicated test database; `prisma migrate reset --force` in
  global setup. Never point tests at a shared dev DB.
- Server components cannot be rendered by RTL. Test their data functions as units and cover
  the rendered result in e2e.

---

## CI

- Run units on every push; e2e on PRs and main.
- Upload traces, videos and reports as artifacts on failure — always.
- Fail the build on a flake retry succeeding, or at minimum surface it. Silent retries hide
  a real race until it happens in production.
