---
name: threat-model
description: Identify what an attacker would target in a feature and what breaks if they succeed. Use before building auth, payments, uploads, or user data. Do NOT use for dependency scanning — that's dependency-audit.
---

# threat-model

## Owns
- Attack surface enumeration for one feature.
- Trust boundaries: where data crosses from less-trusted to more-trusted.
- Abuse cases and risk ranking by plausibility × blast radius.

## Does not own
- Dependency CVEs and licences → `dependency-audit`.
- Secret storage, env vars, config → `secrets-config`.
- Release go/no-go → `prod-readiness-gate`.
- Diff-level security regressions → `core-eng/code-review`.

---

## Step 1 — Scope it to a feature

A threat model of "the whole SaaS" is a document nobody reads. Model **one feature**:
the upload endpoint, the Stripe webhook, the invite flow, the FFI boundary.

If the user asks for a whole-system model, pick the three highest-value features and
model those. Say that you did.

---

## Step 2 — Four questions

Answer these in order. Nothing else belongs in the output.

1. **What are we building?** One paragraph plus a data-flow sketch: who calls what,
   what data moves, where it lands.
2. **What can go wrong?** Trust boundaries (Step 3) plus abuse cases (Step 4).
3. **What do we do about it?** A mitigation per threat, or an explicit accept.
4. **Did we do a good enough job?** What you did not model, and why.

---

## Step 3 — Enumerate trust boundaries

Every point where data crosses from less-trusted to more-trusted is where validation
belongs. In this stack, that is:

| Boundary | Validate |
|---|---|
| tRPC procedure input | Zod schema **and** authorisation — a typed input is not an authorised one |
| Next.js route handler / server action | Body, headers, method, session, tenant |
| Webhook receiver | Signature, timestamp, replay window, idempotency key |
| File upload | Size, real content type (sniff, don't trust the header), filename, storage path |
| Flutter platform channel | Every argument from the Dart side is attacker-controlled on a rooted device |
| Rust FFI / `unsafe` boundary | Pointer validity, length, lifetime, UTF-8, null termination |
| DB query with a user-supplied id | Tenant scoping — see IDOR below |

Mark each boundary in the code you are about to write. If validation lives anywhere other
than at the boundary, an inner caller will eventually skip it.

---

## Step 4 — Abuse cases per surface

Walk only the surfaces the feature actually has.

**Auth** — credential stuffing (no rate limit on login), session fixation (session id not
rotated after login), token replay (no expiry, no revocation list), password reset token
reuse, OAuth state parameter missing, enumeration via distinct error messages.

**Payments** — webhook forgery (signature unverified), replay (same event processed twice
grants two entitlements), race on entitlement (two concurrent checkouts, one subscription),
price/amount trusted from the client, refund not revoking access.

**Uploads** — path traversal in the stored filename, zip bomb / decompression ratio,
content-type spoof (a `.png` that is HTML → stored XSS when served from your origin),
SVG with embedded script, image parser CVEs, unbounded size, serving user files from the
app origin instead of a separate/asset domain.

**Multi-tenant** — **IDOR is the single most common real-world SaaS breach.** Every query
that takes a user-supplied id must filter by tenant in the same statement, not check
afterwards. `where: { id, orgId: ctx.orgId }`, never `findUnique({ id })` then compare.
Also: tenant id taken from the request body instead of the session; admin endpoints with
role checked in the UI only; shared caches keyed without the tenant.

**Client apps (Flutter)** — the binary is fully readable. Anything the client can compute,
an attacker can compute. Client-side checks are UX, never enforcement.

---

## Step 5 — Rank and decide

Rank by **plausibility × blast radius**, not CVSS theatre. A trivially reachable IDOR
leaking every tenant's data outranks a theoretical timing attack with a 9.1 score.

Ship mitigations for the **top 3** now. Log the rest with an owner or an explicit accept.

---

## Step 6 — Output

A table. Not an essay.

```
| Surface        | Threat                            | Mitigation                          | Status  |
|----------------|-----------------------------------|-------------------------------------|---------|
| POST /upload   | content-type spoof → stored XSS   | sniff magic bytes, serve from CDN   | todo    |
| Stripe webhook | replay grants double entitlement  | idempotency on event.id, unique idx | done    |
| getInvoice     | IDOR across tenants               | orgId in the where clause + test    | todo    |
```

Close with one line: **what we did not model and why**. Then stop.

## Sources
Shostack's four-question frame; OWASP Top 10 (A01 broken access control, A07 auth failures).
