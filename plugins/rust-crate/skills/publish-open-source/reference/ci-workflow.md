# CI workflow for a published crate

A working GitHub Actions setup. Adapt the MSRV, the feature names, and the target list; keep the job
structure — each job exists because a specific class of breakage reaches consumers without it.

---

## `.github/workflows/ci.yml`

```yaml
name: ci
on:
  push: { branches: [main] }
  pull_request:

env:
  CARGO_TERM_COLOR: always
  RUSTFLAGS: "-D warnings"
  MSRV: "1.75"

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  test:
    name: test / ${{ matrix.toolchain }} / ${{ matrix.os }}
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        toolchain: [stable]
        include:
          - { os: ubuntu-latest, toolchain: beta }   # catches regressions before they hit stable
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@master
        with: { toolchain: "${{ matrix.toolchain }}" }
      - uses: Swatinem/rust-cache@v2
      - run: cargo test --all-features
      - run: cargo test --doc --all-features       # the README example must actually work

  msrv:
    name: msrv
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@master
        with: { toolchain: "1.75" }                # keep in sync with rust-version
      - uses: Swatinem/rust-cache@v2
      - run: cargo check --all-features
      # An untested MSRV is a fiction. This job is the promise.

  features:
    name: feature matrix
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - uses: Swatinem/rust-cache@v2
      - run: cargo check --no-default-features     # the most common broken configuration
      - run: cargo check --all-features
      - uses: taiki-e/install-action@cargo-hack
      - run: cargo hack check --feature-powerset --depth 2 --no-dev-deps

  lint:
    name: lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with: { components: "clippy, rustfmt" }
      - uses: Swatinem/rust-cache@v2
      - run: cargo fmt --all --check
      - run: cargo clippy --all-targets --all-features -- -D warnings

  docs:
    name: docs
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@nightly
      - uses: Swatinem/rust-cache@v2
      - run: cargo doc --no-deps --all-features
        env:
          RUSTDOCFLAGS: "-D warnings --cfg docsrs"
      # A broken intra-doc link fails the docs.rs build AFTER publish,
      # and cannot be fixed without publishing a new version.

  semver:
    name: semver
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    steps:
      - uses: actions/checkout@v4
      - uses: obi1kenobi/cargo-semver-checks-action@v2

  deny:
    name: cargo-deny
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: EmbarkStudios/cargo-deny-action@v2
        with: { command: check }

  miri:
    name: miri
    runs-on: ubuntu-latest
    # Delete this job if the crate is #![forbid(unsafe_code)].
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@nightly
        with: { components: miri }
      - run: cargo miri test
```

Require `test`, `msrv`, `features`, `lint`, and `docs` as branch protection checks. A check that can
be merged past is documentation, not enforcement.

---

## `.github/workflows/release.yml`

```yaml
name: release
on:
  push:
    tags: ["v*"]

jobs:
  publish:
    runs-on: ubuntu-latest
    environment: crates-io          # require a manual approval before the irreversible step
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - uses: Swatinem/rust-cache@v2

      - name: Tag must match Cargo.toml version
        run: |
          TAG="${GITHUB_REF_NAME#v}"
          MANIFEST=$(cargo metadata --no-deps --format-version 1 \
            | jq -r '.packages[0].version')
          test "$TAG" = "$MANIFEST" || {
            echo "tag $TAG != manifest $MANIFEST"; exit 1; }

      - run: cargo test --all-features
      - run: cargo test --doc --all-features
      - run: cargo publish --dry-run

      - name: Show package contents
        run: cargo package --list
        # Read this in the log before approving. It is the last reversible moment:
        # a published version is public forever, and yank does not delete anything.

      - run: cargo publish
        env:
          CARGO_REGISTRY_TOKEN: ${{ secrets.CARGO_REGISTRY_TOKEN }}

      - uses: softprops/action-gh-release@v2
        with:
          generate_release_notes: false
          body_path: RELEASE_NOTES.md        # the changelog section for this version
```

The `environment:` gate is what makes the publish deliberate. Without it a mistyped tag publishes a
version you cannot delete.

**After the workflow finishes, open `https://docs.rs/crate/<name>/<version>/builds` and confirm the
docs build succeeded.** CI cannot check this for you, and a failure is only fixable with a new
version.

---

## `deny.toml`

```toml
[advisories]
version = 2
ignore = []                     # every entry needs a comment and an expiry date

[licenses]
version = 2
allow = ["MIT", "Apache-2.0", "Apache-2.0 WITH LLVM-exception", "BSD-2-Clause", "BSD-3-Clause", "ISC", "Unicode-3.0"]
confidence-threshold = 0.9

[bans]
multiple-versions = "warn"      # duplicate versions inflate build time and binary size
wildcards = "deny"              # a wildcard version requirement is unpublishable anyway

[sources]
unknown-registry = "deny"
unknown-git = "deny"
```

---

## `rust-toolchain.toml`

Do **not** commit one in a library. It pins every contributor and every CI job to one toolchain,
which defeats the whole matrix above. Pin the toolchain per CI job instead; `rust-toolchain.toml`
belongs in an application, not a published crate.
