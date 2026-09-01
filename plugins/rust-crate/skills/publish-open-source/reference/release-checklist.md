# Release checklist

Work top to bottom. Nothing here is optional for a public crate; the ordering matters because several
steps cannot be undone after `cargo publish`.

---

## A. Before you start

- [ ] Working tree clean, on the release branch, up to date with the remote.
- [ ] CI green on the commit you intend to tag — not on an earlier one.
- [ ] No `TODO`, `FIXME`, `dbg!`, or `unimplemented!` in shipping code:
      ```bash
      rg -n "dbg!\(|todo!\(|unimplemented!\(" src/
      ```

---

## B. API surface

- [ ] `cargo public-api diff --deny=all` reviewed against the last published version.
- [ ] `cargo semver-checks check-release` passes.
- [ ] Version bump matches the largest change in the diff (see `publish-open-source` §4 — a
      behaviour change that no tool detects still counts as breaking).
- [ ] `#[non_exhaustive]` on every public enum and every struct that may grow.
- [ ] No dependency type appears in the public API unless deliberately version-locked or `pub use`d.
- [ ] `#[must_use]` on builders and pure transforms.
- [ ] Reviewed by `critique/api-critique` if the surface changed meaningfully.

---

## C. Documentation

- [ ] `cargo doc --no-deps --all-features` builds with **no warnings**:
      ```bash
      RUSTDOCFLAGS="-D warnings --cfg docsrs" cargo +nightly doc --no-deps --all-features
      ```
      A broken intra-doc link fails the docs.rs build *after* publish and cannot be fixed without a
      new version.
- [ ] `#![warn(missing_docs)]` present, and no `missing_docs` warnings.
- [ ] Crate-root `//!` shows a working example in the first screen.
- [ ] `cargo test --doc` passes with all features.
- [ ] README example copy-pasted into a scratch project and run — the actual test, not the doctest.
- [ ] `## Features` section documents every feature flag.
- [ ] README states install, example, docs link, MSRV, licence, and maintenance intent.

---

## D. Build matrix

```bash
cargo build --no-default-features
cargo build --all-features
cargo test  --all-features
cargo +$MSRV check --all-features          # e.g. cargo +1.75 check
cargo clippy --all-targets --all-features -- -D warnings
cargo fmt --check
cargo deny check
cargo miri test                            # if the crate contains any unsafe
```

- [ ] All green.
- [ ] `rust-version` in `Cargo.toml` matches the MSRV job that actually ran.

---

## E. Metadata

- [ ] `version` bumped in `Cargo.toml` (and in every workspace member released with it).
- [ ] `description`, `license`, `repository`, `documentation`, `readme`, `keywords` (≤5),
      `categories` (valid crates.io slugs), `rust-version` all present.
- [ ] `[package.metadata.docs.rs]` sets `all-features = true` and the `docsrs` cfg.
- [ ] `CHANGELOG.md` has the new version, dated, with Breaking / Added / Changed / Fixed sections
      written for consumers.
- [ ] Both licence files present if dual-licensed.

---

## F. Package contents — the last reversible moment

```bash
cargo package --list
```

- [ ] Read the **entire** list. Nothing unintended: no `.env`, no credentials, no large fixtures, no
      `target/`, no personal notes.
- [ ] Package size is sane (`ls -l target/package/*.crate`). A multi-megabyte crate usually means a
      missing `exclude`.
- [ ] Secret scan of the packaged tree:
      ```bash
      rg -n "(api[_-]?key|secret|password|token|BEGIN [A-Z ]*PRIVATE KEY)" \
        $(cargo package --list | sed 's|^|.|')
      ```

**A published version is public forever.** `cargo yank` does not delete anything. This step is the
only one that prevents a permanent leak.

```bash
cargo publish --dry-run
```

- [ ] Dry run succeeds.

---

## G. Publish

```bash
git tag -a v0.4.0 -m "v0.4.0"
git push origin main --follow-tags
cargo publish
```

- [ ] Tag pushed **before** publishing, so a shipped version always maps to a commit.
- [ ] For a workspace: publish leaf crates first, waiting for the index between each — a dependent
      published too early fails to resolve.

---

## H. After publish — do not skip

- [ ] **Open the docs.rs build log** (`https://docs.rs/crate/<name>/<version>/builds`) and confirm it
      succeeded. A failed docs.rs build cannot be retried by re-publishing the same version; the only
      fix is a new patch release.
- [ ] `cargo add <crate>@<version>` in a scratch project, then compile and run the README example
      against the published artifact — not the local path.
- [ ] Create the GitHub release from the tag, with the changelog section as the body.
- [ ] Open a `## [Unreleased]` heading in `CHANGELOG.md` for the next cycle.

---

## I. If something is wrong after publish

1. `cargo yank --version X.Y.Z` — stops new dependents selecting it. Existing lockfiles still use it.
2. Fix, bump the patch version, publish immediately. **A yank without a replacement strands
   consumers.**
3. Note the yank and its reason in the changelog.
4. If a secret was published: **rotate the credential now.** Yanking does not remove the file, and
   crates.io does not delete versions. See `secure-ship/secrets-config`.
