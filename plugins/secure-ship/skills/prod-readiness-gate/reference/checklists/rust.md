# Ship checklist — Rust crate (public / crates.io)

`cargo publish` is **irreversible**. A published version can be yanked, never replaced.
Treat every item here as blocking unless stated otherwise.

## Publish mechanics
- [ ] `cargo publish --dry-run` clean.
- [ ] `cargo package --list` reviewed — nothing secret, nothing huge, nothing accidental.
      Fix with `include`/`exclude` in `Cargo.toml`, not by hoping.
- [ ] Version bumped and the git tag matches it.
- [ ] `CARGO_REGISTRY_TOKEN` comes from CI secrets, never a shared machine's credentials file.
- [ ] Crate name is one you are willing to own permanently — squatting rules make it final.

## Docs
- [ ] `cargo doc --no-deps` builds with **zero warnings**; `#![deny(rustdoc::broken_intra_doc_links)]`.
- [ ] docs.rs will build it — features and target-specific code declared in
      `[package.metadata.docs.rs]` if the default build would fail there.
- [ ] **Every public item documented.** Enforce with `#![warn(missing_docs)]`.
- [ ] Crate-level doc explains what it is for, the shortest useful example, and every env
      var or file path the crate touches.
- [ ] README examples compile — they are doctests (`#![doc = include_str!("../README.md")]`)
      or they will rot within two releases.
- [ ] Feature flags documented, including which are default and what each pulls in.

## Correctness
- [ ] `cargo test` green, including doctests.
- [ ] `cargo clippy --all-targets --all-features -- -D warnings` clean.
- [ ] `cargo fmt --check` clean.
- [ ] Tested with `--no-default-features` and with each feature enabled — a feature-gated
      compile break is the most common post-publish bug report.
- [ ] `unsafe` blocks: each has a `// SAFETY:` comment stating the invariant that holds.
      If the crate is `unsafe`-free, `#![forbid(unsafe_code)]` says so.
- [ ] Miri run on `unsafe`-containing tests, if any.

## Compatibility
- [ ] **MSRV declared** (`rust-version` in `Cargo.toml`) **and tested in CI** on that exact
      toolchain. An undeclared MSRV means every consumer's CI is your CI.
- [ ] **Semver reviewed against the previous release.** Adding a public enum variant, a
      trait method without a default, or a struct field to a non-`#[non_exhaustive]` public
      struct is a breaking change. `cargo semver-checks` if available.
- [ ] Platform support claimed in the README is actually built in CI (Windows/Linux at minimum).
- [ ] Dependency versions are ranges (`1.2`), not pins — pinning in a library forces
      duplicate versions on consumers.

## Legal & metadata
- [ ] `license = "MIT OR Apache-2.0"` (the ecosystem norm) with **both** `LICENSE-MIT` and
      `LICENSE-APACHE` files present in the package.
- [ ] No GPL/AGPL in the dependency tree — blocking for a public crate.
- [ ] `cargo deny check` clean and wired into CI.
- [ ] `cargo audit` clean of criticals.
- [ ] `Cargo.toml` complete: `description`, `repository`, `documentation`, `keywords`,
      `categories`, `readme`. A crate without these is invisible on crates.io.
- [ ] `CHANGELOG.md` updated with this version, dated, breaking changes called out first.

## Release
- [ ] Tagged commit is the one being published; working tree clean.
- [ ] Post-publish: `cargo add <crate>` in a scratch project and compile the README example
      against the real published version. This catches packaging mistakes nothing else does.
