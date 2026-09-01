---
name: publish-open-source
description: Prepare and publish a Rust crate to crates.io — docs, README, licensing, semver, MSRV, CI, changelog, release process. Use before any publish. Do NOT use for the general prod readiness gate, that's secure-ship/prod-readiness-gate.
---

# publish-open-source

## Owns
- The publish pipeline, and everything an external consumer judges the crate by.

## Does not own
- The go/no-go decision → `secure-ship/prod-readiness-gate` (that skill *calls* this one's checklist).
- API design → `api-ergonomics`. API review → `critique/api-critique`.
- Dependency risk → `secure-ship/dependency-audit`.

---

## 1. Documentation is the product

**For a public crate, docs.rs is where adoption is won or lost.** A stranger arrives with a problem,
spends about thirty seconds deciding whether this crate solves it, and leaves. Undocumented items, a
one-sentence crate root, and an example that does not compile all resolve that the same way.

- **`#![warn(missing_docs)]`** at minimum, `deny` once the surface is clean — the only mechanism that
  stops documentation decaying as the crate grows.
- **A crate-level `//!` doc explaining what this is, with a working example in the first screen.**
  Not a feature list — the shortest useful program someone can copy.
- **Doc examples on every significant public item.** A signature plus one sentence is documented in
  name only.
- **`#![doc = include_str!("../README.md")]`** so README and crate docs cannot diverge, plus
  `cfg_attr(docsrs, feature(doc_cfg))` for feature annotations.

**Doc examples are tests.** They must pass in CI. This is what prevents the classic "the README
example doesn't work" failure — the most common way a crate loses a user who was already sold. Hide
setup lines with `#`; never use `ignore`, which is untested and will rot.

---

## 2. `Cargo.toml` completeness

```toml
[package]
name = "mycrate"
version = "0.4.0"
edition = "2021"
rust-version = "1.75"                      # MSRV, and it is a promise
description = "One line, no marketing, says what it does."
license = "MIT OR Apache-2.0"              # the ecosystem default; ship both LICENSE files
repository = "https://github.com/you/mycrate"
documentation = "https://docs.rs/mycrate"
readme = "README.md"
keywords = ["http", "client", "api"]       # max 5, each max 20 chars
categories = ["web-programming::http-client"]   # from the fixed crates.io list
exclude = ["/.github", "/benches/data", "/assets", "/fuzz"]

[package.metadata.docs.rs]
all-features = true
rustdoc-args = ["--cfg", "docsrs"]
```

- **A missing `description` or `license` blocks publish**; missing `keywords`/`categories` just makes
  the crate unfindable, which is a quieter failure.
- **`exclude` keeps the package small** — check what actually ships with `cargo package --list`.
- `[package.metadata.docs.rs]` with `all-features` is what makes docs.rs show feature-gated items.
  Without it, half your API is invisible on the page people actually read.

---

## 3. MSRV

**Declare it (`rust-version`), test it in CI with a pinned toolchain job, and treat raising it as a
minor-version event with a changelog note.**

An undeclared MSRV means the minimum is "whatever the newest feature you accidentally used requires",
moving silently with every commit; consumers on a fixed toolchain discover it as a build failure.
Pick a policy and write it in the README — "N-2 stable releases" or "whatever is in current Debian
stable" are both defensible. `cargo msrv` finds the real minimum if you have not tracked it.

---

## 4. Semver, taken seriously

**Breaking changes are not just signature changes.** Each of these is a major version:

- Adding a variant or field to anything not `#[non_exhaustive]` — including adding a private field to
  a struct with public fields, which breaks literal construction.
- Adding a required trait method (no default body); adding or tightening a bound; making an optional
  parameter required.
- Removing or renaming anything public, including a re-export.
- **Bumping the major version of a dependency whose types appear in your public API.**
- Losing a `Send`/`Sync`/`Copy` impl — including by accident, by adding an `Rc` to a private field.
- Adding a blanket impl, or a generic method that removes object safety.

```bash
cargo semver-checks check-release      # mechanical violations, in CI on every PR
cargo public-api diff --deny=all       # the full surface diff, reviewed by a human
```

**These tools do not catch behaviour changes.** A function that starts returning an error where it
used to succeed is breaking, and no tool will tell you. Route the release through
`critique/api-critique` for that pass.

Before 1.0, `0.x.y` → `0.(x+1).0` is the breaking bump. **Reaching 1.0 is a commitment, not a
milestone** — do not do it until the API has survived a real consumer.

---

## 5. Changelog

**`CHANGELOG.md` in Keep a Changelog format, updated in the same PR as the change** — not
reconstructed from `git log` at release time, which produces a list of commit subjects nobody can
use.

```markdown
## [Unreleased]

## [0.4.0] - 2026-08-31
### Breaking
- `Client::new` now returns `Result`; call `?` or `.expect(..)`. (#42)
### Added
- `Client::timeout` for a per-request deadline. (#38)
### Fixed
- Retries no longer double-count the first attempt. (#41)
```

Write it **for consumers** — what changed for them and what they must do. "Refactored the internal
buffer" belongs in the commit. Every breaking entry names the migration.

---

## 6. CI matrix

| Job | Why |
|---|---|
| `stable` — build, test, doctest | The baseline. |
| `beta` | Catches a regression before it reaches your users on stable. |
| **MSRV pinned toolchain** | The declared minimum is otherwise fiction. |
| `--no-default-features` | The most common broken configuration. |
| `--all-features` | The second most common. |
| `clippy --all-targets --all-features -- -D warnings` | Warnings that are not errors accumulate. |
| `cargo fmt --check` | Ends formatting discussion permanently. |
| `cargo deny check` | Licences, advisories, duplicate versions. |
| `cargo doc --no-deps` with `-D warnings` and `--cfg docsrs` | **A broken intra-doc link fails the docs.rs build after publish**, and cannot be fixed without a new version. |
| `cargo semver-checks` | Mechanical breakage, before it is published. |

See `reference/ci-workflow.md` for a working workflow.

---

## 7. Release sequence

Run it in this order, every time — `reference/release-checklist.md` is the copy-paste form.

1. **Version bump** in `Cargo.toml` (and every workspace member that moves with it).
2. **Changelog** — move `Unreleased` to the new version with the date.
3. **`cargo publish --dry-run`** — it builds the package exactly as crates.io will.
4. **`cargo package --list` and read it.** Verify nothing unintended ships: no `.env`, no fixtures,
   no keys, no `target/`. This is the last moment you can look.
5. **Tag** the release commit (`v0.4.0`) and push the tag.
6. **`cargo publish`.**
7. **Verify docs.rs built successfully** — check the build log on docs.rs, not just that the page
   exists. **A docs.rs build failure cannot be retried by re-publishing the same version**; you must
   publish a new patch version to fix it. This is the most common post-publish regret.

For a workspace, publish leaf crates first and wait for the index to update between each — a
dependent crate published too early fails to resolve.

---

## 8. Yanking is not deletion

**A published version is public forever.** `cargo yank` stops new dependents selecting it; it does
not remove the files, and anyone with a lockfile keeps building it.

Therefore: **check for secrets in the package before publishing, not after.** A key in a test fixture
shipped inside a `.crate` is public permanently, and the only remediation is rotating the credential.
Step 4 above is that check — see `secure-ship/secrets-config`.

Yank when a version is genuinely broken or unsound, publish the fix immediately, and say why in the
changelog. Yanking without a replacement strands consumers.

---

## 9. The contribution surface

What a stranger sees before they read a line of your code:

- **README**, in this order: one line on what it is, `cargo add mycrate`, a working example, a
  docs.rs link, MSRV, licence. The example goes above the badges.
- **`CONTRIBUTING.md`** — how to run the tests, what CI requires, whether you want PRs. Issue
  templates, and a `SECURITY.md` with a reporting address if the crate is security-relevant.
- **Both licence files** (`LICENSE-MIT`, `LICENSE-APACHE`) if you dual-license.

**State the maintenance intent honestly.** If this is a crate you wrote for one project and will not
maintain, say so in the README's first section. **Abandoned crates with confident READMEs waste other
people's time** — someone will build on it, file an issue, and get silence. "Maintained for my own
use; issues welcome but responses may be slow" costs you one sentence and saves them a week.
