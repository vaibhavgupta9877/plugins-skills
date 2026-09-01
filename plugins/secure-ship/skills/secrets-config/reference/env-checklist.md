# Env checklist — per-stack templates

Load this when adding or auditing environment variables. Adapt to the repo's existing
conventions; do not impose these files on a repo that already has a working scheme.

---

## Next.js (T3) — typed schema

`src/env.js` (or `.ts`):

```ts
import { createEnv } from "@t3-oss/env-nextjs";
import { z } from "zod";

export const env = createEnv({
  server: {
    NODE_ENV: z.enum(["development", "test", "production"]),
    DATABASE_URL: z.string().url(),
    AUTH_SECRET: z.string().min(32),
    STRIPE_SECRET_KEY: z.string().startsWith("sk_"),
    STRIPE_WEBHOOK_SECRET: z.string().startsWith("whsec_"),
  },
  client: {
    // public identifiers only — inlined into the browser bundle forever
    NEXT_PUBLIC_APP_URL: z.string().url(),
  },
  runtimeEnv: {
    NODE_ENV: process.env.NODE_ENV,
    DATABASE_URL: process.env.DATABASE_URL,
    AUTH_SECRET: process.env.AUTH_SECRET,
    STRIPE_SECRET_KEY: process.env.STRIPE_SECRET_KEY,
    STRIPE_WEBHOOK_SECRET: process.env.STRIPE_WEBHOOK_SECRET,
    NEXT_PUBLIC_APP_URL: process.env.NEXT_PUBLIC_APP_URL,
  },
  emptyStringAsUndefined: true,
});
```

Rules:
- Constrain the shape (`.url()`, `.startsWith("sk_")`, `.min(32)`), not just `z.string()`.
  A truncated key should fail at build, not at the first charge.
- Never add a `.default()` to a secret. A defaulted secret is a silent misconfiguration.
- Import `env` everywhere; `process.env` appears only inside `runtimeEnv`.

`.env.example` — committed, every key, every value blank:

```
NODE_ENV=
DATABASE_URL=
AUTH_SECRET=
STRIPE_SECRET_KEY=
STRIPE_WEBHOOK_SECRET=
NEXT_PUBLIC_APP_URL=
```

---

## Flutter

Compile-time values are **public**. Use `--dart-define-from-file` for build config
(base URLs, flavour names, public SDK ids) — never for anything privileged.

`config/dev.json` (committed), `config/prod.json` (committed — public values only):

```json
{ "API_BASE_URL": "https://api.example.com", "FLAVOR": "prod", "SENTRY_DSN": "https://...@o0.ingest.sentry.io/0" }
```

```
flutter build appbundle --dart-define-from-file=config/prod.json
```

```dart
const apiBaseUrl = String.fromEnvironment('API_BASE_URL');
```

User tokens at rest:

```dart
const storage = FlutterSecureStorage(
  aOptions: AndroidOptions(encryptedSharedPreferences: true),
);
await storage.write(key: 'refresh_token', value: token);
```

Desktop config path — never beside the executable:

```dart
final dir = await getApplicationSupportDirectory();
final file = File(p.join(dir.path, 'settings.json'));
```

Signing material (`key.jks`, `key.properties`, provisioning profiles) is gitignored and
lives in CI secrets, base64-encoded, written to disk at build time and deleted after.

---

## Rust

Binaries read env; libraries take config as parameters.

```rust
// src/config.rs — binary only
pub struct Config { pub database_url: String, pub port: u16 }

impl Config {
    pub fn from_env() -> anyhow::Result<Self> {
        Ok(Self {
            database_url: std::env::var("APP_DATABASE_URL")
                .context("APP_DATABASE_URL must be set")?,
            port: std::env::var("APP_PORT").unwrap_or_else(|_| "8080".into()).parse()?,
        })
    }
}
```

- Prefix every var with the crate/app name. Document each one in the crate docs.
- Redact secrets in `Debug`:

```rust
impl std::fmt::Debug for Config {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        f.debug_struct("Config").field("database_url", &"[redacted]")
         .field("port", &self.port).finish()
    }
}
```

- Before publishing: `cargo package --list | grep -Ei 'env|secret|key|credential'`.

---

## Pre-commit secret scan

Cheap and catches the common cases. `.git/hooks/pre-commit` (or via `lefthook`/`husky`):

```sh
#!/bin/sh
set -e

# 1. Never commit a real env file
if git diff --cached --name-only | grep -E '(^|/)\.env($|\.[^e])' ; then
  echo "BLOCKED: .env staged. Use .env.example instead."; exit 1
fi

# 2. High-signal key shapes in staged content
if git diff --cached -U0 | grep -nE \
  'sk_live_[A-Za-z0-9]{16,}|whsec_[A-Za-z0-9]{16,}|ghp_[A-Za-z0-9]{20,}|AKIA[0-9A-Z]{16}|-----BEGIN [A-Z ]*PRIVATE KEY-----' ; then
  echo "BLOCKED: looks like a live secret. Rotate it if it was ever pushed."; exit 1
fi

# 3. Signing material
if git diff --cached --name-only | grep -E '\.(jks|keystore|p12|pem|mobileprovision)$|key\.properties' ; then
  echo "BLOCKED: signing material staged."; exit 1
fi
```

A hook is a safety net, not a control — it is bypassable with `--no-verify` and it runs
only on machines that installed it. Pair it with a server-side scan (GitHub secret
scanning / `gitleaks` in CI) which is not bypassable.

---

## Audit pass — run through this list

- [ ] `.env` gitignored; `.env.example` committed with every key, all values blank.
- [ ] Every var in the code appears in the typed schema and in `.env.example`.
- [ ] No `NEXT_PUBLIC_*` holds anything from the server-secret class.
- [ ] No secret compiled into a Flutter build; privileged calls proxied via backend.
- [ ] Shipped SDK keys are restricted at the provider (bundle id / fingerprint / quota).
- [ ] Rust: every env var read is prefixed and documented; none in the library layer.
- [ ] CI secrets masked; no secret echoed in a build log or baked into an image layer.
- [ ] `git log -p` / secret scanner shows no historical leak. Anything found → rotate.
