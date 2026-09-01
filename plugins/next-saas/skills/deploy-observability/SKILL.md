---
name: deploy-observability
description: Configure deployment, environments, CI, monitoring, and error tracking for the Next.js SaaS on Vercel or a portable alternative. Use for deploy config, not for release gating — that's secure-ship/prod-readiness-gate.
---

# deploy-observability

## Owns
- Deploy configuration, environments, CI pipeline, migration strategy.
- Error tracking, logging, alerting, DB hosting and pooling.

## Does not own
- Go/no-go release gating → `secure-ship/prod-readiness-gate`.
- Front-end performance → `web-ui/web-performance`.
- Secret values and env policy → `secure-ship/secrets-config`.

---

## 1. Three environments, minimum

| Environment | Database | Purpose |
|---|---|---|
| **Preview** (per PR) | A branch/ephemeral DB, or a shared dev DB | Review the change in a real deploy |
| **Staging** | **Its own database** | Rehearse migrations and releases |
| **Production** | Production | Real users |

**Preview deployments must never touch the production database.** This is a common and expensive
mistake: a preview build runs `migrate deploy` or a seed script, or a reviewer clicks "delete" on
what they think is test data. On Vercel, environment variables are per-environment — set
`DATABASE_URL` for Preview explicitly rather than letting it inherit.

Neon/PlanetScale-style database branching per preview is the clean answer where available. Otherwise
a single shared dev database is acceptable and production is not.

---

## 2. CI before deploy

**A pipeline that only builds catches nothing.** Minimum, on every PR:

```
typecheck  →  lint  →  unit + integration tests  →  build
```

- `tsc --noEmit` as its own step — `next build` does not typecheck everything you think it does.
- `prisma migrate diff` against the target to show what the deploy will run, in the PR.
- `prisma generate` before typecheck, or the types are stale.
- Cache `node_modules` and `.next/cache`; a slow pipeline gets bypassed.
- Block merge on failure. A red pipeline that merges anyway is decoration.

---

## 3. Migrations on deploy

- **`prisma migrate deploy` as an explicit release step**, before the new code serves traffic — not
  at runtime import, and never `db push`.
- **Expand → contract for anything breaking**, so a rollback does not lose data:
  1. Add the new column, nullable. Deploy.
  2. Deploy code writing both old and new.
  3. Backfill in batches.
  4. Deploy code reading the new one.
  5. Drop the old column in a *later* release.
- Every step must be compatible with the code currently running, because during a deploy both
  versions are live.
- Review lock duration on large tables before shipping — see `trpc-prisma-patterns`.
- **Know the rollback.** Code rollback is redeploying the previous build. Schema rollback usually is
  not possible: that is why expand-then-contract exists.

---

## 4. Vercel lock-in map

The user is on Vercel and will move if something better appears. **Name the cost of every coupled
decision at the time it is made.**

| Portable | Coupled to Vercel |
|---|---|
| Next.js build output (`output: "standalone"` → Docker) | **Vercel KV / Blob / Postgres** wrappers |
| Prisma + any Postgres | **Vercel Cron** |
| Auth.js | **Edge Middleware specifics** (runtime, geo/IP headers) |
| Stripe | **ISR / `revalidateTag` semantics** at the platform level |
| tRPC, React, Tailwind | **Function timeout and memory behaviour** |
| Standard `fetch` caching | Image Optimization as a hosted service |

Keep the exit cheap:
- **`output: "standalone"`** and a working `Dockerfile` in the repo from day one. It is a few lines,
  and it is the difference between a week's migration and a quarter's.
- **Standard Postgres** (Neon, Supabase, RDS) over a platform-branded wrapper. Same protocol, movable.
- Keep cron logic in a route handler that any scheduler can call, with a shared-secret header — then
  the scheduler is swappable.
- Prefer Node runtime over Edge unless Edge earns it; Edge code is the least portable thing you write.

**When the user reaches for a Vercel-only primitive, say what the exit cost is, then build it.** The
call is theirs; the information is yours to supply.

---

## 5. Error tracking

- Wired for **server, client, and edge** — three separate initialisations, and the edge one is the
  one people forget.
- **Source maps uploaded** in CI, or production stack traces are unreadable minified noise.
- **Release tagging** with the commit SHA, so an error points at a diff.
- **Verified with a real test event from a production build.** "Sentry is installed" is not evidence.
- Scrub PII in `beforeSend` — request bodies, tokens, emails, card data.
- Set a sample rate deliberately; 100% tracing on a busy app is expensive and mostly noise.

---

## 6. Logging

**`console.log` is not observability.** Structured JSON, one event per line, with:

- A **request id** propagated through the whole request (generate at the edge, pass through context,
  return it in the error response so a user can quote it in a support ticket).
- `userId` / `orgId` where present, so a report becomes a query.
- Event name, duration, and outcome — not prose.
- **Never** tokens, passwords, card data, full request bodies, or PII.

Log at boundaries: request in/out, external call in/out, job start/end. Logging inside every function
produces volume, not insight.

---

## 7. Alerts on what matters only

**Alerts nobody acts on train people to ignore alerts.** Start with these five and add only with
evidence:

1. Error-rate spike (relative to baseline, not an absolute count).
2. Failed payments — a business event, not just an error.
3. Auth failure spike — credential stuffing in progress.
4. Database connection saturation.
5. Deploy failed, or migration failed.

Every alert names the runbook step. An alert with no action is a notification, and belongs in a
dashboard instead.

---

## 8. Serverless Postgres

**Serverless exhausts database connections by default** — each concurrent function invocation opens
its own, and a modest traffic spike hits the connection limit while the database is otherwise idle.

Use a pooler: **PgBouncer** (transaction mode), **Prisma Accelerate**, or the provider's built-in
pooling (Neon, Supabase). Point `DATABASE_URL` at the pooled endpoint and `DIRECT_URL` at the direct
one for migrations — Prisma needs both, and migrations cannot run through a transaction-mode pooler.

Transaction-mode pooling disables prepared statements; add `pgbouncer=true` to the connection string
so Prisma adapts.

---

## 9. Backups

- Automated, with a **stated retention window**.
- **A restore has actually been performed once**, into a scratch database, and timed. An untested
  backup is a hypothesis.
- Point-in-time recovery for anything holding customer data.
- Know the RPO and RTO and write them down — "how much data can we lose" and "how long to be back"
  are product decisions, not infrastructure trivia.
