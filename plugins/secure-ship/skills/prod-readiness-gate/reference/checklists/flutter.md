# Ship checklist — Flutter (Android · Windows · Linux)

Each item: PASS with evidence, BLOCK with the failure, or UNVERIFIED.

## Real devices — not emulators
- [ ] Release build (`--release`, not profile or debug) installed and exercised on a **real
      low-end Android device** — 2–3GB RAM, older SoC. Emulators hide startup crashes,
      jank, ABI mistakes, and thermal behaviour.
- [ ] Tested on the oldest `minSdkVersion` you claim to support.
- [ ] Windows build launched on a **clean machine** with no Flutter/VS toolchain installed —
      catches missing VC++ redistributables and DLLs that only exist on the dev box.
- [ ] Linux build launched on a clean target distro — catches missing GTK/`libsecret`/
      shared-library assumptions.
- [ ] Cold start after an update (not a fresh install) does not crash — this is where a
      changed local DB schema or shared-prefs key blows up on real users.

## State & data
- [ ] Local DB/prefs migration path tested from the **previous shipped version**, with real data.
- [ ] Offline start, mid-session network loss, and slow/flaky network all handled with a
      visible state — no infinite spinner, no silent empty screen.
- [ ] No unbounded local cache; a way to clear app data without reinstalling.
- [ ] Background/foreground transitions do not lose in-progress user input.

## Security
- [ ] **No secrets in the binary.** No API keys, no backend credentials, no signing
      material. Privileged calls go through the backend. (See `secrets-config`.)
- [ ] Shipped third-party keys (Maps, Firebase) restricted at the provider by package name
      + SHA-1 fingerprint.
- [ ] User tokens in `flutter_secure_storage`; nothing sensitive in `SharedPreferences`.
- [ ] Certificate/TLS behaviour is default — no disabled validation left from debugging.
- [ ] Obfuscation enabled where relevant: `--obfuscate --split-debug-info=<dir>`, and the
      symbol files are **archived** or your crash reports are unreadable.
- [ ] Desktop config written to the OS app-data dir, not next to the executable.

## Permissions & store
- [ ] Permission list minimised — every entry in the manifest justified by a real feature.
      Remove ones a removed feature left behind; Play review rejects unexplained ones.
- [ ] Sensitive permissions have an in-app rationale shown before the system prompt.
- [ ] Store metadata complete: description, screenshots at required sizes, feature graphic,
      content rating, target audience, data-safety form filled honestly.
- [ ] **Privacy policy exists and is reachable at a live URL** — Play blocks without it.
- [ ] Package name / app id final. It cannot be changed after the first publish.
- [ ] `versionCode` incremented; `versionName` matches the changelog.

## Build & signing
- [ ] Release signed with the **upload key**, not a debug key; keystore backed up somewhere
      you will still have access to in three years.
- [ ] Play App Signing enrolled and understood.
- [ ] App bundle (`.aab`) for Play; APK only for direct distribution.
- [ ] R8/ProGuard rules keep anything reflected (JSON models, native plugin classes) —
      verify on the release build, not debug.
- [ ] App size checked; no accidental large asset shipped.
- [ ] Windows/Linux artifacts are versioned and reproducible from a tagged commit.

## Runtime quality
- [ ] Crash/error reporting live in the release build, verified with a forced test crash.
- [ ] No `print`/`debugPrint` on shipped paths; no debug banners; no dev endpoints.
- [ ] No jank on the heaviest list/scroll on the low-end device (profile mode, real device).
- [ ] Deep links / intent filters tested if used.
- [ ] Accessibility pass: text scaling at 200% does not clip, tap targets ≥48dp,
      screen reader reaches primary actions.
