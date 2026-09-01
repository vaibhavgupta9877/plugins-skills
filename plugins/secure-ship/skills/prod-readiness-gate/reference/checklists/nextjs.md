# Ship checklist — Next.js SaaS (Vercel-primary)

Each item: PASS with evidence, BLOCK with the failure, or UNVERIFIED.

## Auth & session
- [ ] Session cookie is `httpOnly`, `secure`, `sameSite=lax` (or `strict`), with a sane maxAge.
- [ ] Session id rotates on login and on privilege change.
- [ ] Logout invalidates server-side, not just client-side.
- [ ] Password reset / magic-link tokens are single-use, short-lived, and constant-time compared.
- [ ] OAuth callback validates `state`; redirect targets are allow-listed, not reflected from the query.
- [ ] Login, signup, reset, and invite errors do not enumerate accounts.

## Tenant scoping — the default failure
- [ ] **Every** tRPC procedure and route handler touching tenant data filters by tenant
      **in the query**, not after the fetch. Walk the whole router list, not a sample.
- [ ] Tenant id comes from the session, never from the request body or a header.
- [ ] Admin/elevated procedures check role server-side (UI gating is not a check).
- [ ] There is at least one automated test that asserts a cross-tenant read returns 404/403.
- [ ] Caches (Redis, `unstable_cache`, SWR keys) include the tenant in the key.
- [ ] Signed URLs / file paths cannot be walked to another tenant's objects.

## Rate limiting & abuse
- [ ] Rate limits on login, signup, password reset, invite, and every unauthenticated write.
- [ ] Rate limits on expensive authenticated endpoints (search, export, report generation).
- [ ] Limits keyed by IP **and** account; verified by actually tripping one.
- [ ] Upload endpoints enforce max size and real content type (sniffed, not header-trusted).

## Payments (Stripe)
- [ ] Webhook signature verified with the signing secret, on the raw body.
- [ ] Webhook handler is idempotent — unique constraint on `event.id`, replays are no-ops.
- [ ] Handler returns 2xx fast and does slow work out-of-band; Stripe retries are safe.
- [ ] Entitlement derives from Stripe's state, never from a client-supplied price/plan.
- [ ] Cancel, refund, chargeback, and failed-renewal paths revoke access.
- [ ] Test-mode keys are not in the production env.

## Headers & transport
- [ ] CSP set and actually enforced (not report-only) — no blanket `unsafe-inline` script.
- [ ] `Strict-Transport-Security`, `X-Content-Type-Options: nosniff`, `Referrer-Policy`,
      `X-Frame-Options`/`frame-ancestors`, `Permissions-Policy`.
- [ ] CORS allow-list is explicit; no `*` on credentialed endpoints.
- [ ] User-uploaded files served from a separate origin/CDN, not the app origin.

## Data
- [ ] Automated backups on, retention stated — **and someone has actually restored one.**
- [ ] Migrations reversible, or one-way with a written reason and a fresh verified backup.
- [ ] Migration is backwards compatible with the currently deployed code (expand → migrate → contract).
- [ ] PII columns identified; deletion/export path exists if you claim GDPR/DPDP compliance.
- [ ] Connection pooling correct for serverless (a pooler, not one connection per invocation).

## Observability
- [ ] Sentry (or equivalent) wired for server, client, **and** edge; source maps uploaded;
      a real event from a production build has arrived.
- [ ] Structured logs with a request id; no tokens, passwords, card data, or full bodies logged.
- [ ] Uptime check on a real endpoint that touches the DB, not a static `/health` that always 200s.
- [ ] Alerts route somewhere a human sees at 3am.

## Performance
- [ ] Core Web Vitals measured on real hardware and a throttled connection — not a
      localhost Lighthouse run. LCP, INP, CLS within budget on the slowest real page.
- [ ] No N+1 on the main dashboard query; the heaviest page's query count is known.
- [ ] Images through `next/image`; fonts self-hosted or preloaded; no render-blocking third-party script.
- [ ] Bundle size checked after the last dependency addition.

## Deploy
- [ ] Production env vars set in Vercel for the production environment specifically.
- [ ] Preview deployments do not point at the production database.
- [ ] Rollback = redeploy previous deployment; confirmed the previous build still exists.
- [ ] Custom domain, SSL, and `NEXT_PUBLIC_APP_URL` all agree.
