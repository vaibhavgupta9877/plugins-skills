---
name: secrets-config
description: Handle secrets, environment variables, and configuration safely across Next.js, Flutter, and Rust. Use when adding any key, token, or env-dependent setting. Do NOT use for auth flow design — that's next-saas/auth-and-billing.
---

# secrets-config

## Owns
- Where a secret is allowed to live, per stack.
- Env var schemas, validation at boot, config layering.
- Key rotation, and what must never reach a client bundle.

## Does not own
- Auth flow and session design → `next-saas/auth-and-billing`.
- Dependency risk → `dependency-audit`.
- Attack surface analysis → `threat-model`.

---

## Rule 0 — Classify before you place

Every value is one of three things. Place it by class, not by habit.

- **Public config** — API base URL, feature flag, analytics site id. Shippable anywhere.
- **Server secret** — DB URL, Stripe secret key, webhook signing secret, service tokens.
  Server process env only. Never in a client bundle, never in a mobile binary.
- **User credential** — the end user's session token or refresh token. Belongs in OS-backed
  secure storage on the device, not in your source or your env.

If you cannot say which class a value is, ask. Guessing puts server secrets in bundles.

---

## Flutter — stated bluntly

**There is no such thing as a secret in a shipped Flutter binary.** API keys, signing
secrets, backend credentials, `--dart-define` values, values in `assets/`, strings in the
Dart snapshot — all extractable from the APK/AAB/exe with standard tooling. Obfuscation
raises the cost by minutes, not by orders of magnitude.

Consequences, non-negotiable:

- Any design that relies on client-side secrecy is **broken**, not "risky". Redesign it.
- Route every privileged call through your backend. The app authenticates as *the user*;
  the backend holds the keys and enforces what that user may do.
- Third-party SDK keys that must ship (Firebase, Maps, Sentry DSN) are not secrets — they
  are public identifiers. Secure them with server-side restrictions: allowed bundle id,
  SHA-1 fingerprint, referrer, quota, and rules. Assume they will be copied.
- `flutter_secure_storage` protects **user** tokens at rest on the device (Keychain /
  EncryptedSharedPreferences). It does **not** protect **your** keys, because your code
  must be able to read them and so can an attacker with the same binary.
- Desktop (Windows/Linux): user config and tokens live in the OS app-data directory —
  `%APPDATA%\<Company>\<App>\` on Windows, `$XDG_CONFIG_HOME/<app>` (default
  `~/.config/<app>`) on Linux. Use `path_provider`'s
  `getApplicationSupportDirectory()`. **Never** write config next to the executable:
  under `C:\Program Files` it is unwritable or silently virtualised, and on Linux it
  breaks any packaged install.

---

## Next.js

- **`NEXT_PUBLIC_*` is public, permanently, to everyone.** It is inlined into the client
  bundle at build time. There is no un-shipping it. Before adding one, ask whether it is
  in the *public config* class above.
- **Validate env at boot with a typed schema** (`@t3-oss/env-nextjs` + Zod), splitting
  `server:` and `client:`. A missing or malformed var must fail the **build**, not
  production at 3am. The schema is also the documentation of what this app needs.
- Server-only modules get `import "server-only"` at the top, so an accidental client
  import becomes a build error instead of a leaked key.
- Never read `process.env` scattered through the codebase. Import from the validated
  `env` object so there is exactly one place where a var name is spelled.
- Secrets live in the platform's env store (Vercel project env, per-environment). Not in
  `next.config.js`, not in a committed `.env`, not in the repo at all.
- Middleware and edge runtime have their own env plumbing — verify a secret is actually
  available there before relying on it.

---

## Rust

- No secrets in `Cargo.toml`, no secrets in source, no secrets in doctests or examples,
  no secrets in test fixtures. `cargo package` ships more than people expect — check
  `cargo package --list` before publishing.
- For a public crate: **never read env vars with surprising names.** Every env var the
  crate reads must be documented in the crate-level docs, prefixed with the crate name,
  and safe to be absent.
- Secrets come in as parameters from the caller, not read from the ambient environment by
  a library. Libraries take config; binaries read env.
- `crates.io` tokens live in CI secrets only (`CARGO_REGISTRY_TOKEN`), never in
  `~/.cargo/credentials` on a shared machine and never echoed in a CI log.
- Do not log a secret-bearing struct. Implement `Debug` manually or wrap in a type whose
  `Debug` prints `[redacted]` (e.g. `secrecy::Secret`).

---

## Universal rules

- **Never commit `.env`.** Add it to `.gitignore` before the first commit, not after.
- **`.env.example` is committed**, with every key present and every value blank. It is the
  onboarding contract. A new key without a matching `.env.example` line is an incomplete
  change — call it out in review.
- One env file per environment; no shared "staging uses prod's DB" shortcuts.
- Config layering: defaults in code → env file → process env → explicit CLI/arg override.
  Later wins. Document the order once.
- Secrets are passed to CI as masked secrets, never as build args that land in an image
  layer or a build log.

---

## Rotation — the part people skip

**Any secret that has ever appeared in a git commit, a log line, a CI output, a
screenshot, a Slack paste, or an error message is burned. Rotate it.**

Removing it from `HEAD` does nothing: the object is still in history, in every clone, in
every fork, and in whatever scraper indexed the repo within minutes of the push.

Procedure, in this order:
1. **Rotate first** — issue a new key at the provider, deploy it, verify traffic works.
2. **Revoke the old key** at the provider. This is the step that actually fixes it.
3. Then, optionally, scrub history (`git filter-repo`) and force-push — cosmetic cleanup,
   not remediation.
4. Check the provider's access logs for use of the leaked key while it was live.

---

## Before you finish

Read `reference/env-checklist.md` when you are adding or auditing env vars — it has the
per-stack schema templates and a pre-commit secret-scan hook.

Then verify: the new key is in `.env.example` blank · it is in the typed schema · nothing
`NEXT_PUBLIC_` holds a server secret · nothing secret is compiled into a Flutter build ·
`.env` is gitignored · the value never appeared in a commit.
