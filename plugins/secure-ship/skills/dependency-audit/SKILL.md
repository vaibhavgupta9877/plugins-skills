---
name: dependency-audit
description: Audit third-party dependencies for vulnerabilities, licence conflicts, abandonment, and bloat across npm, pub.dev, and crates.io. Do NOT use for version pinning at publish time — that's rust-crate/publish-open-source.
---

# dependency-audit

## Owns
- The add/don't-add decision for a new dependency.
- CVE scanning, licence compatibility, maintenance health, transitive bloat.
- Separating real risk from scanner noise.

## Does not own
- Version pinning and semver policy for publishing → `rust-crate/publish-open-source`.
- Secret storage → `secrets-config`.
- Release go/no-go → `prod-readiness-gate`.

---

## Before adding a dependency — four questions

1. **Is it maintained?** Last release date, commit activity, open-to-closed issue ratio,
   whether a security issue from six months ago is still open. One maintainer with no
   releases in 18 months is an abandonment risk regardless of star count.
2. **What does it drag in?** Check the transitive count before installing, not after:
   `npm info <pkg> dependencies`, the crates.io dependency tab, the pub.dev page. A
   left-pad-shaped package with 40 transitive deps is a supply-chain surface, not a utility.
3. **What's the licence?** Yours and every transitive one. See the licence rules below.
4. **Could this be 30 lines of our own code?** Most small utility deps fail this question.
   Debounce, slugify, clamp, a date format — write it, test it, own it. Reserve
   dependencies for things that are genuinely hard: crypto, parsers, protocol clients,
   database drivers, platform integration.

Never write your own crypto. Question 4 does not apply there.

---

## Tooling

```
# npm / pnpm
pnpm audit --prod            # dev-only findings are noise for a release decision
pnpm outdated
pnpm why <pkg>               # who pulled this in

# Rust
cargo audit                  # RUSTSEC advisories
cargo deny check             # licences, bans, duplicate versions, advisories
cargo tree -d                # duplicate versions of the same crate

# Flutter / Dart
dart pub outdated
flutter pub deps             # full transitive tree
```

Wire `cargo deny check` into CI for **any public crate** — licence and advisory drift is
otherwise invisible until a consumer reports it.

---

## Licences

**Public Rust crates: default to `MIT OR Apache-2.0`.** That is the ecosystem norm and it
is what consumers' own `cargo deny` configs expect.

**A GPL or AGPL transitive dependency makes the crate unusable by most consumers — treat
it as blocking, not advisory.** The same applies to any proprietary SaaS project that
distributes binaries. Do not "note it and move on"; find a replacement or stop.

Also blocking: no licence at all (default is all-rights-reserved — you have no right to
use it), or a licence that changed under you (BSL/SSPL relicensing of a formerly-Apache
package is common and is a real migration event).

Fine: MIT, Apache-2.0, BSD-2/3, ISC, Zlib, Unlicense, MPL-2.0 (file-level copyleft — safe
to link, do not modify their files without publishing the change).

---

## Flutter specifics

**Audit native plugin dependencies too.** A pub package is a thin wrapper; the risk lives
in the Android/iOS/desktop code it pulls in.

- A pub package that wraps an abandoned Android library **is** an abandoned dependency,
  no matter how recent its Dart-side release.
- Check the plugin's `android/build.gradle` and podspec for pinned old native SDK versions
  — these break on the next `compileSdk` bump and block Play Store submissions.
- Check platform coverage honestly: many popular plugins are mobile-only and will fail the
  Windows/Linux desktop build. Verify before adopting, not at release.
- Prefer federated plugins with a `flutter.dev` / `dart.dev` publisher for anything
  touching platform APIs.

---

## Reporting — distinguish risk from noise

Escalating everything trains the user to ignore you.

**Blocking:** exploitable CVE in a runtime path with attacker-reachable input · GPL/AGPL
in a distributed artifact · unmaintained crypto or auth dependency · no licence.

**Report, don't block:** CVE in a dev-only or build-only transitive dependency · advisory
with no available patch and no reachable code path · duplicate versions inflating bundle
size · outdated-but-fine majors.

State reachability explicitly: "`postcss` advisory is dev-only (build tooling), not in the
shipped bundle — not a release blocker." That sentence is the whole value of the audit.

Output format: one table — package · version · finding · severity · reachable? · action.
Then a single line naming the one dependency you would remove if you could remove one.
