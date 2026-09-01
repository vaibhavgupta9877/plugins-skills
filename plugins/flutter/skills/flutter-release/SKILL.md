---
name: flutter-release
description: Build, sign, package, and distribute Flutter apps for Android, Windows, and Linux — store submission, installers, CI, versioning. Use for shipping, not for the go/no-go decision — that's secure-ship/prod-readiness-gate.
---

# flutter-release

## Owns
- Build configuration, flavours, signing, packaging, store and distribution mechanics, release CI.

## Does not own
- Whether the release should happen at all → `secure-ship/prod-readiness-gate`.
- Runtime platform behaviour → `flutter-desktop-platform`.
- Secret values and env policy → `secure-ship/secrets-config`.

---

## 1. Flavours and per-environment config

Three flavours: `dev`, `staging`, `prod`, each with its own application id / package name so all
three can be installed side by side.

**Configuration comes in at build time via `--dart-define-from-file`, never from a committed config
file.**

```bash
flutter build appbundle --flavor prod --dart-define-from-file=config/prod.json
```

- `config/*.json` is gitignored; CI writes it from secrets. Commit a `config/example.json` with the
  key names and dummy values so the shape is discoverable.
- Read values with `String.fromEnvironment` — it is const-folded, so a missing key silently becomes
  `""`. Assert on startup that every required key is non-empty, in dev and staging.
- **Nothing in a `--dart-define` is secret.** It is compiled into a binary the user holds and can be
  extracted with `strings`. API base URLs and feature flags: yes. Any credential that grants access:
  no — see `secure-ship/secrets-config`.

---

## 2. Android

- **`.aab` (app bundle) for Play**, `.apk` only for direct distribution and testing.
- **Keystore lives in CI secrets only, never in the repo.** Base64 the `.jks` into a secret, decode
  it at build time, and pass the passwords as secrets too. Back up the keystore somewhere you will
  still have it in five years — losing it means you cannot ever update the app under that listing.
  `key.properties` is gitignored.
- **`--obfuscate --split-debug-info=build/symbols`** on every release build, with the symbol
  directory **archived as a CI artifact per version**. Without those symbols a production crash
  report is unreadable, and you cannot regenerate them later.
- **R8/ProGuard keep rules for anything using reflection** — most JSON serialisation that is not
  code-generated, and several native SDKs. The failure is a release-only `ClassNotFoundException`
  that never appears in debug. Test the release build, not just the debug one.
- Target SDK compliance deadlines are enforced by Play annually; an app that misses one stops being
  updatable and eventually stops being discoverable. Track the date.
- Play Console: data-safety form (must match what the app actually collects), content rating, and a
  privacy policy URL. A wrong data-safety declaration is a policy violation, not a form error.
- Ship through internal → closed → open testing tracks; staged rollout percentages for production so
  a bad release can be halted.

---

## 3. Windows

- **MSIX (`msix` package)** for Store submission or modern sideload; Inno Setup or WiX/MSI when you
  need per-machine install, custom actions, or file associations on older setups.
- **Code signing: state the cost plainly.** An unsigned Windows binary triggers a SmartScreen
  "unrecognised app" warning that a majority of non-technical users will not click through. **Budget
  for an OV or EV certificate** (EV gets reputation immediately; OV accumulates it over installs and
  time). This is not optional polish — it materially determines install rate.
- Sign with `signtool` in CI using a certificate from a secret store or a cloud HSM. Timestamp the
  signature (`/tr`), or the binary stops validating when the certificate expires.
- **Bundle the correct VC++ redistributable** or the app fails to start on a clean machine with a
  missing-DLL dialog. Verify on a fresh VM.
- Ship the whole `build/windows/x64/runner/Release/` directory — the exe alone will not run.
- File associations, Start menu entry, and uninstall entry are registered by the installer; decide
  whether the app is per-user or per-machine before you build the installer.

---

## 4. Linux — choose the channel deliberately

| Channel | Reach | Cost |
|---|---|---|
| **`.deb`** | Debian/Ubuntu only | Simplest. You declare dependencies; apt resolves them. |
| **Flatpak** | Broadest, all distros | Sandbox constraints must be designed around — file access, portals, autostart. Flathub review. |
| **Snap** | Ubuntu-centric | Similar confinement issues; slower first start; a single store. |
| **AppImage** | Portable, no install | No integration (menu entry, updates, associations) unless you add it; must bundle nearly everything. |

- **Bundle shared libraries or declare them correctly.** The default failure is an app that runs on
  the dev machine and dies with a missing `.so` on a clean install. Check with `ldd` on the built
  binary and reconcile the list against what you declare or ship.
- **Build on the oldest glibc you intend to support** — glibc is forward- but not backward-compatible,
  so a binary built on a current distro will not start on an older LTS. An old container image in CI
  is the usual answer.
- Ship a `.desktop` file, an icon at multiple sizes, and (for `.deb`) correct `Depends:`.
- Pick one channel to support properly rather than four badly. `.deb` plus AppImage covers most
  users with the least work; Flatpak if you want Flathub distribution.

---

## 5. Versioning

**One source of truth: `pubspec.yaml`'s `version: x.y.z+build`.** Every platform derives from it —
Android `versionName`/`versionCode`, Windows MSIX version, `.deb` version.

- **Build numbers must increase monotonically for Play.** A rejected upload because the build number
  was reused is a routine, avoidable delay. Derive it from the CI run number or the commit count,
  never by hand.
- MSIX requires a four-part `a.b.c.d` version and rejects a repeated one; map `+build` to the fourth
  part.
- Tag the release commit (`v1.4.2`) and build from the tag, so a shipped binary maps to a diff.

---

## 6. Release CI

**Build all three targets on every release tag.** Windows and Linux need their own runners —
`windows-latest` and `ubuntu-latest`. A pipeline that only runs on Linux silently stops testing the
Windows build, and you find out when you try to ship.

The shape:

```
tag v* → [android, windows, linux] in parallel
         each: flutter pub get → analyze → test → build → sign → upload artifact
       → archive debug symbols → create GitHub release → submit to stores
```

Gate the release job on tests passing. Cache the pub and Gradle caches — an uncached Android release
build is slow enough that people start skipping CI. See `reference/packaging.md` for commands and a
workflow template.

---

## 7. Crash reporting

- Initialised **before `runApp`**, wrapping it, plus `FlutterError.onError` and
  `PlatformDispatcher.instance.onError` so both framework and raw async errors are captured.
- **Symbol upload wired into CI for all three platforms** — Android `--split-debug-info` output,
  and the desktop equivalents. Uploading manually means it eventually stops happening.
- **Verify with a deliberate test crash from a release build**, once per platform. "Sentry is in
  `pubspec.yaml`" is not evidence that crashes arrive.
- Tag events with the version and flavour, or you cannot tell which release regressed.
- Scrub PII before send.

---

## 8. Updates

**Play handles Android. Desktop has no default — decide it rather than shipping nothing.**

- Store-distributed (MSIX via the Microsoft Store, Snap, Flatpak) updates itself. Simplest, if you
  accept the store's review.
- Self-updating (`auto_updater`/Sparkle-style, or your own check-and-download) — you own the update
  server, signature verification of the downloaded artifact, and rollback.
- Manual download — acceptable for internal tools, hostile for consumer apps.

Whichever you choose, ship a **version check with a minimum-supported-version response from your API
on day one**, even if the update itself is manual. Without it you cannot force off a build with a
data-corrupting bug, and old clients hit your API forever.
