# Plugin: `secure-ship`

**Enable for:** every project, always.
**Purpose:** the difference between "it works on my machine" and "it can hold real users' data." Security and release gating.
**Skills:** 4

**Honest framing for the agent:** a generic "be secure" skill is worthless — the model already says "validate input." These four skills only earn their slot by being *specific to this stack set and checkable*. Every rule must be something you can point at in a file and mark pass/fail. Delete any rule you can't verify.

---

## `threat-model` — SUPPORTING

**description:** `Identify what an attacker would target in this feature or system, and what breaks if they succeed. Use before building anything handling auth, payments, uploads, or user data. Do NOT use for dependency scanning — that's dependency-audit.`

**Owns:** attack surface enumeration, trust boundaries, abuse cases, risk ranking.
**Does NOT own:** dependency CVEs → `dependency-audit`; secret storage → `secrets-config`; release gating → `prod-readiness-gate`.

**Body must cover:**
- Scope to the feature, not the universe. A threat model of "the whole SaaS" is a document nobody reads.
- Four questions: what are we building, what can go wrong, what do we do about it, did we do a good enough job.
- Enumerate trust boundaries explicitly — every point where data crosses from less-trusted to more-trusted is where validation belongs. In this stack that's: tRPC procedure inputs, Next.js route handlers, Flutter platform channels, Rust FFI boundaries, webhook receivers, file uploads.
- Abuse cases per surface: auth (credential stuffing, session fixation, token replay), payments (webhook forgery, replay, race on entitlement), uploads (path traversal, zip bomb, content-type spoof, stored XSS), multi-tenant (IDOR — the single most common real-world SaaS breach).
- Rank by *plausibility × blast radius*, not by CVSS theatre. Ship mitigations for the top 3; log the rest.
- Output: a short table — surface · threat · mitigation · status. Not an essay.

**Budget:** 130 lines.

---

## `secrets-config` — CORE

**description:** `Handle secrets, environment variables, and configuration safely across Next.js, Flutter, and Rust. Use when adding any key, token, or env-dependent setting. Do NOT use for auth flow design — that's next-saas/auth-and-billing.`

**Owns:** secret storage, env var schemas, config layering, key rotation, what must never reach a client bundle.
**Does NOT own:** auth logic → `next-saas/auth-and-billing`; dependency risk → `dependency-audit`.

**Body must cover:**
- **The Flutter rule, stated bluntly:** there is no such thing as a secret in a shipped Flutter binary. Anything compiled into the app — API keys, signing secrets, backend credentials — is extractable. Any design that relies on client-side secrecy is broken. Route privileged calls through a backend. `flutter_secure_storage` protects *user* tokens at rest on device; it does not protect *your* keys.
- **The Next.js rule:** `NEXT_PUBLIC_*` is public, permanently, to everyone. Validate env at boot with a typed schema (`@t3-oss/env-nextjs` + Zod) so a missing var fails the build, not production. Server-only modules get `import "server-only"`.
- **The Rust rule:** no secrets in `Cargo.toml`, no secrets in source, no secrets in doctests. For public crates, never read env vars with surprising names — document every one. `crates.io` tokens live in CI secrets only.
- Never commit `.env`. `.env.example` is committed with every key present and every value blank.
- Rotation: any secret that has ever appeared in a git commit, a log, a CI output, or a screenshot is burned. Rotate it. Removing it from HEAD does nothing.
- Desktop Flutter (Windows/Linux): config lives in the OS app-data directory, not next to the executable. Name the correct paths.

**reference/:** `env-checklist.md` — per-stack env schema template and pre-commit hook.

**Budget:** 170 lines.

---

## `dependency-audit` — SUPPORTING

**description:** `Audit third-party dependencies for vulnerabilities, licence conflicts, abandonment, and bloat across npm, pub.dev, and crates.io. Use before adding a dependency or before a release.`

**Owns:** CVE scanning, licence compatibility, maintenance-health judgement, transitive bloat.
**Does NOT own:** version pinning for publish → `rust-crate/publish-open-source`.

**Body must cover:**
- Before adding any dependency, four questions: is it maintained (last release, open issue ratio), how many transitive deps does it drag, what's the licence, and could this be 30 lines of our own code instead. Most small utility deps fail question four.
- Tooling: `npm audit` / `pnpm audit`, `cargo audit` + `cargo deny`, `dart pub outdated`. Wire `cargo deny` into CI for any public crate.
- **Licence rule for public Rust crates:** default to `MIT OR Apache-2.0` (the Rust ecosystem norm). A GPL/AGPL transitive dependency makes the crate unusable by most consumers — treat it as blocking, not advisory.
- Flutter: audit native plugin dependencies too. A pub package that wraps an abandoned Android library is an abandoned dependency.
- Distinguish real risk from noise. A CVE in a dev-only transitive dependency is not a release blocker; report it as such rather than escalating everything.

**Budget:** 110 lines.

---

## `prod-readiness-gate` — CORE

**description:** `Run the pre-release checklist for a Flutter app, Rust crate, or Next.js SaaS before shipping. Use before any production deploy, store submission, or crate publish. Do NOT use during feature development.`

**Owns:** go/no-go release gating, per-stack ship checklists.
**Does NOT own:** the mechanics of building/signing artifacts → `flutter/flutter-release`, `rust-crate/publish-open-source`, `next-saas/deploy-observability`.

**Body must cover:**
- This is a **gate**, not advice. Output is PASS or BLOCK with the specific failing items. Do not soften a BLOCK.
- Universal: no secrets in the bundle · error tracking wired and verified with a test event · no `console.log`/`print`/`dbg!` in shipped paths · dependency audit clean of criticals · rollback plan exists and has been described · migrations are reversible or explicitly one-way with a documented reason.
- **Next.js SaaS:** auth session handling verified · every tRPC procedure that touches tenant data enforces tenant scoping (IDOR is the default failure here) · rate limiting on auth and write endpoints · webhook signature verification · Stripe webhook idempotency · CSP and security headers set · DB has backups and someone has restored one · Core Web Vitals measured on real hardware.
- **Flutter:** release build tested on a real low-end Android device, not just an emulator · Windows and Linux builds actually launched on clean machines · permissions minimised and justified in store listing · offline and no-network states handled · app doesn't crash on cold start after update · obfuscation enabled where relevant · store metadata and privacy policy present.
- **Rust crate:** `cargo publish --dry-run` clean · docs build on docs.rs · every public item documented · README examples compile (doctests) · MSRV declared and tested in CI · semver reviewed against previous release · `CHANGELOG.md` updated · licence files present.
- End with the single highest-risk item and what it would cost if it fails.

**reference/:** `checklists/nextjs.md`, `checklists/flutter.md`, `checklists/rust.md` — the full expanded lists, loaded only for the relevant stack.

**Budget:** 150 lines (body); checklists live in `reference/`.
