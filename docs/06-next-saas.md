# Plugin: `next-saas`

**Enable for:** Next.js T3 SaaS projects only.
**Purpose:** application architecture and backend for the SaaS. Not UI (`web-ui`), not motion (`animate`).
**Skills:** 5

**Stack:** Next.js App Router · TypeScript · tRPC · Prisma · Auth.js (NextAuth) · Tailwind · Vercel-primary hosting.
**Portability note the agent must respect:** the user is on Vercel but has said they'll move if something better appears. Every deploy-coupled decision must be flagged where it creates lock-in, and a portable alternative named. Do not architect for Vercel-only primitives without saying so.

---

## `t3-architecture` — CORE

**description:** `Set up, structure, or restructure a T3 Next.js application — folder layout, server/client boundaries, env validation, layering conventions. Do NOT use for UI composition, that's web-ui/web-design-engineering.`

**Owns:** project scaffolding, directory conventions, layering, server/client boundary policy, env schema wiring, removing create-t3-app boilerplate.
**Does NOT own:** UI components → `web-ui/*`; tRPC procedure design → `trpc-prisma-patterns`; auth flows → `auth-and-billing`.

**Body must cover:**
- Fresh setup: what to delete from `create-t3-app` immediately (example router, example post model, unused providers), and what to add before the first feature (env schema, error boundary, logger, base layout, token system hook into `web-ui/design-system`).
- Directory convention: feature-oriented, not type-oriented. `src/features/<feature>/{ui,server,schema.ts}` beats `src/components` + `src/utils` sprawl. `src/server/api/routers` for tRPC, `src/server/db.ts` for the Prisma singleton.
- **Server/client boundary policy, stated as a rule:** server components by default, `"use client"` pushed to the smallest leaf that needs interactivity. Prisma is never importable from a client component — enforce with `import "server-only"` in `src/server/**`.
- Env validation via `@t3-oss/env-nextjs` + Zod, split `server` and `client` schemas, build fails on missing vars. Cross-reference `secure-ship/secrets-config`.
- The "utils" trap: no `utils.ts` dumping ground. Every helper lives next to its feature or in a named module.
- Prisma client singleton pattern for dev hot-reload; never instantiate per request.
- When restructuring an existing app, propose an incremental migration path, not a rewrite. State the cost honestly if a rewrite is genuinely cheaper.

**Absorb:** `/mnt/skills/user/t3-next-setup`, structural portions of `/mnt/skills/user/t3-frontend-craft`.

**Budget:** 190 lines.

---

## `trpc-prisma-patterns` — CORE

**description:** `Write tRPC routers and Prisma queries — procedure design, input validation, authorization middleware, transactions, N+1 avoidance, migrations. Do NOT use for public API surface review, that's critique/api-critique.`

**Owns:** procedure implementation, Prisma schema and queries, middleware, transactions, migrations, data access patterns.
**Does NOT own:** auth session setup → `auth-and-billing`; API shape critique → `critique/api-critique`; deployment DB config → `deploy-observability`.

**Body must cover:**
- Procedures model **use-cases, not tables**. `createInvoiceAndNotify` beats `invoice.create` + three client calls. A router that mirrors your Prisma models one-to-one is a design smell.
- Every procedure has a Zod input schema that accepts exactly what it uses — no pass-through objects, no `z.any()`.
- **Authorization is middleware, not per-procedure code.** Define `protectedProcedure` and, critically, a tenant/org-scoped procedure that injects the scope. Then state the rule bluntly: **every query touching tenant data must filter by the scope from the session, never from client input.** Accepting an `orgId` from the client is the IDOR bug that breaks real SaaS products. Make this the loudest rule in the skill.
- Errors: `TRPCError` with correct codes (`UNAUTHORIZED`, `FORBIDDEN`, `NOT_FOUND`, `CONFLICT`, `BAD_REQUEST`). Never leak internal messages or stack traces to the client. Log the detail server-side, return the safe message.
- Prisma correctness: `select`/`include` explicitly — never fetch whole rows by default; `findUniqueOrThrow` where absence is exceptional; `$transaction` for multi-write invariants; `Promise.all` for independent reads.
- N+1: the default failure. Use nested `include`, or `findMany` with `in`. Enable Prisma query logging in dev and actually read it.
- Migrations: `migrate dev` locally, `migrate deploy` in CI. Never `db push` against production. Every migration must be reviewed for lock duration on large tables; adding a non-null column without a default locks the table.
- Indexes: add them with the query that needs them, in the same PR.
- Optimistic updates and cache invalidation on the client side of tRPC — state the invalidation policy so stale UI doesn't ship.

**reference/:** `prisma-patterns.md` — transaction, pagination (cursor over offset), soft-delete, and multi-tenant schema patterns.

**Budget:** 200 lines.

---

## `auth-and-billing` — CORE

**description:** `Implement authentication, sessions, and subscription billing for the SaaS — Auth.js flows, Stripe integration, entitlements, webhooks. Do NOT use for secret storage policy, that's secure-ship/secrets-config.`

**Owns:** auth flows, session strategy, password handling, Stripe subscriptions, webhooks, entitlement checks, plan gating.
**Does NOT own:** secrets storage → `secure-ship/secrets-config`; threat modelling → `secure-ship/threat-model`; procedure-level authz middleware → `trpc-prisma-patterns`.

**Body must cover:**
- Auth.js setup: session strategy choice (database sessions for revocability, JWT for edge/statelessness) with the tradeoff stated — do not pick silently. Prisma adapter wiring.
- Credentials provider: `argon2id` or `bcrypt` (cost ≥ 12), constant-time comparison, generic error messages that don't reveal whether an email exists, rate limiting on login and reset, email verification before granting access, tokens that are single-use and expire.
- Session security: httpOnly + secure + sameSite cookies, rotation on privilege change, explicit logout invalidation.
- **Stripe integration rules, each stated as non-negotiable:**
  - The webhook is the source of truth for subscription state, never the client redirect.
  - Verify the webhook signature. An unverified webhook endpoint is a free-subscription button.
  - Webhooks are idempotent — store processed `event.id` and no-op on repeat. Stripe *will* retry.
  - Entitlements are read from your own DB, never queried from Stripe on the request path.
  - Handle the full lifecycle: `checkout.session.completed`, `customer.subscription.updated`, `.deleted`, `invoice.payment_failed`. Missing `payment_failed` means users keep paid access for free.
  - Test with the Stripe CLI locally before shipping.
- Plan gating: a single `hasEntitlement(user, feature)` check used on both server and client. Client-side gating is UX only — the server check is the real one.
- Never trust price or plan IDs from the client.

**reference/:** `stripe-lifecycle.md` — event handling table and idempotency store schema.

**Budget:** 200 lines.

---

## `api-design` — SUPPORTING

**description:** `Design the shape of new API surfaces — tRPC router structure, REST route handlers, webhooks, error contracts, versioning. Do NOT use to review an existing API, that's critique/api-critique.`

**Owns:** designing new surfaces before implementation.
**Does NOT own:** implementation → `trpc-prisma-patterns`; review → `critique/api-critique`.

**Body must cover:**
- Design the surface from the consumer's use-cases backwards. Write the call site first, then build the procedure that makes it read well.
- Router structure mirrors product domains, nested at most two levels deep.
- When to use a REST route handler instead of tRPC: webhooks, third-party callbacks, file upload/download, anything consumed by a non-TS client. Everything else is tRPC.
- Uniform error contract across the whole surface — one shape, typed, documented.
- Pagination: cursor-based by default. Offset pagination breaks on live data and gets slow.
- Idempotency: any mutation that could be retried needs an idempotency key. State which ones.
- Versioning: for internal tRPC, none — change the client together. For anything external, version from day one; retrofitting versioning is expensive.
- Long-running work does not belong in a request. Name the queue/background approach and its Vercel constraint (function timeouts) explicitly.

**Budget:** 110 lines.

---

## `deploy-observability` — SUPPORTING

**description:** `Configure deployment, environments, CI, monitoring, and error tracking for the Next.js SaaS on Vercel or a portable alternative. Use for deploy config, not for release gating — that's secure-ship/prod-readiness-gate.`

**Owns:** deploy config, environments, CI pipeline, error tracking, logging, alerting, DB hosting.
**Does NOT own:** go/no-go gating → `secure-ship/prod-readiness-gate`; frontend perf → `web-ui/web-performance`.

**Body must cover:**
- Three environments minimum: preview (per-PR), staging with a separate database, production. Preview deployments must never touch the production DB — this is a common and expensive mistake.
- CI before deploy: typecheck, lint, unit tests, build. A pipeline that only builds catches nothing.
- Migration strategy on deploy: `prisma migrate deploy` as a release step, not at runtime import. Expand-then-contract for breaking schema changes so a rollback doesn't lose data.
- **Vercel lock-in map** — name what's portable and what isn't: portable (Next.js build output, Prisma, Auth.js, Stripe); coupled (Vercel KV/Blob/Cron, Edge Middleware specifics, ISR revalidation semantics, function timeout behaviour). Recommend Docker-buildable output and standard Postgres so a move stays cheap. If the user reaches for a Vercel-only primitive, say what the exit cost is.
- Error tracking wired with source maps, release tagging, and a verified test event. Untested error tracking is no error tracking.
- Structured logging with request/trace correlation. `console.log` is not observability.
- Alerts on what matters only: error rate spike, failed payments, auth failures, DB connection saturation. Alerts nobody acts on train people to ignore alerts.
- Connection pooling for serverless Postgres (PgBouncer / Prisma Accelerate / Neon pooling) — serverless exhausts connections by default.
- Backups exist and a restore has actually been performed once.

**Budget:** 140 lines.
