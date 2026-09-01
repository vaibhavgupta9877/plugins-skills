---
name: auth-and-billing
description: Implement authentication, sessions, and subscription billing for the SaaS — Auth.js flows, Stripe integration, entitlements, webhooks. Do NOT use for secret storage policy, that's secure-ship/secrets-config.
---

# auth-and-billing

## Owns
- Auth.js configuration, session strategy, credentials handling, email verification.
- Stripe subscriptions, webhooks, entitlements, plan gating.

## Does not own
- Secret storage and env policy → `secure-ship/secrets-config`.
- Threat modelling → `secure-ship/threat-model`.
- Procedure-level authz middleware and tenant scoping → `trpc-prisma-patterns`.

---

## 1. Session strategy — choose out loud

**Do not pick silently.** State the tradeoff and the choice:

| | Database sessions | JWT sessions |
|---|---|---|
| Revocation | Immediate — delete the row | Not until expiry, unless you add a denylist |
| Cost | A DB read per request | None |
| Edge/middleware | Needs a DB reachable from edge | Works anywhere |
| Role/plan changes | Reflected immediately | Stale until refresh |

**Default to database sessions for a SaaS.** Revocability is worth the read: "log out all devices",
"remove a member", and "this account is compromised" all become real operations rather than
best-effort. Choose JWT when you need edge middleware without a database, and then design for the
staleness — short expiry plus a refresh, and never gate a paid feature on a JWT claim alone.

Wire the Prisma adapter; put `orgId`/`role` into the session callback so `trpc-prisma-patterns`'
`orgProcedure` has a scope to inject.

---

## 2. Credentials provider

Only if you actually need passwords — OAuth and email links avoid this entire surface.

- **`argon2id` (preferred) or `bcrypt` with cost ≥ 12.** Never SHA-anything, never a homemade scheme.
- **Constant-time comparison**, and hash a dummy password when the user does not exist so the timing
  does not reveal account existence.
- **Generic errors**: "Incorrect email or password" for both cases. The signup and reset flows must
  not reveal existence either — reset always says "if that address exists, we've sent a link".
- **Rate limit login, signup, and password reset** — by IP *and* by account. Without this, credential
  stuffing is trivial and free. Lock or add friction after repeated failures.
- **Email verification before granting access** to anything meaningful.
- **Reset and verification tokens: single-use, short-lived (≤1h), high-entropy, stored hashed**, and
  invalidated on use and on password change. A reset token that survives its own use is a takeover.
- Enforce a minimum length (12+) rather than composition rules, and check against a breached-password
  list if you can.
- Changing a password invalidates all other sessions.

---

## 3. Session security

- Cookies: `httpOnly`, `secure`, `sameSite: "lax"`, scoped path, sensible `maxAge`.
- **Rotate the session id on login and on privilege change.** Without rotation, session fixation is
  live.
- Logout invalidates server-side, not just by clearing the cookie.
- Consider a shorter absolute lifetime for sessions that can spend money.
- CSRF: Auth.js handles its own routes; **your own POST route handlers are your responsibility** —
  `sameSite` plus an origin check, or a token.

---

## 4. Stripe — non-negotiables

**1. The webhook is the source of truth for subscription state. Never the client redirect.**
The user closing the tab after paying must still get access; the user reaching `/success` by typing
the URL must not.

**2. Verify the webhook signature, on the raw body.**
An unverified webhook endpoint is a free-subscription button — anyone can POST a
`checkout.session.completed`. In App Router, read the raw text; do not let a body parser touch it:

```ts
export async function POST(req: Request) {
  const raw = await req.text();
  const sig = (await headers()).get("stripe-signature")!;
  let event: Stripe.Event;
  try {
    event = stripe.webhooks.constructEvent(raw, sig, env.STRIPE_WEBHOOK_SECRET);
  } catch {
    return new Response("Invalid signature", { status: 400 });
  }
  // ...
}
```

**3. Webhooks are idempotent. Stripe *will* retry, and events can arrive out of order.**
Store the processed `event.id` with a unique constraint and no-op on repeat:

```ts
try {
  await db.stripeEvent.create({ data: { id: event.id, type: event.type } });
} catch {
  return new Response("ok", { status: 200 });   // already processed
}
```
Also guard ordering: ignore an update whose `created` is older than the state you already stored.

**4. Entitlements are read from your own database, never queried from Stripe on the request path.**
A Stripe API call in a page render is a latency and availability dependency on someone else's uptime.
The webhook writes your DB; your app reads your DB.

**5. Handle the full lifecycle.** Missing `payment_failed` means users keep paid access for free:

| Event | Do |
|---|---|
| `checkout.session.completed` | Link customer + subscription to the org; grant entitlement |
| `customer.subscription.updated` | Plan changes, upgrades, downgrades, cancel-at-period-end |
| `customer.subscription.deleted` | Revoke entitlement at period end |
| `invoice.payment_failed` | Mark past-due, notify, start dunning, revoke after grace |
| `invoice.payment_succeeded` | Clear past-due, extend the period |

**6. Never trust price, plan, or amount from the client.** Pass a price *id* you control, look it up
server-side, and let Stripe compute the amount. Store the `orgId` in checkout `metadata` and
`client_reference_id` so the webhook knows who paid.

**7. Test with the Stripe CLI locally** (`stripe listen --forward-to`, `stripe trigger`) before
shipping. Exercise the failure events, not just the happy path.

Handle the race explicitly: two concurrent checkouts for one org must not produce two subscriptions —
a unique constraint on `(orgId)` in your subscription table, plus reconciliation.

See `reference/stripe-lifecycle.md` for the handler skeleton and the idempotency schema.

---

## 5. Plan gating

**One function, used on both sides:**

```ts
export function hasEntitlement(sub: Subscription | null, feature: Feature): boolean { … }
```

- **Client-side gating is UX only.** It hides a button. The **server check is the real one**, and it
  belongs in the procedure that does the work — not in the component that calls it.
- Enforce quantitative limits (seats, projects, API calls) at write time, inside the same transaction
  as the write. A check-then-write races and lets a user exceed the limit.
- Grace over cliff: past-due keeps read access and blocks writes, for a stated number of days, with a
  clear in-app message. Instant lockout on a failed card generates support load and churn.
- When a downgrade puts a user over the new limit, define what happens — read-only, or archived —
  rather than discovering it in production.
- Every gate returns `PRECONDITION_FAILED` with the plan needed, so the UI can offer the upgrade
  instead of a dead end.
