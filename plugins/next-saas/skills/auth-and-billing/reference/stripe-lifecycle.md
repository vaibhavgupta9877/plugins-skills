# Stripe lifecycle

The handler skeleton, the idempotency store, and the event table. Adapt the model names; keep the
guarantees.

---

## Schema

```prisma
model StripeEvent {
  id          String   @id            // Stripe's event.id — the idempotency key
  type        String
  processedAt DateTime @default(now())
}

model Subscription {
  id                 String   @id @default(cuid())
  orgId              String   @unique          // one subscription per org; blocks the double-checkout race
  stripeCustomerId   String   @unique
  stripeSubId        String   @unique
  priceId            String
  status             SubStatus
  seats              Int      @default(1)
  currentPeriodEnd   DateTime
  cancelAtPeriodEnd  Boolean  @default(false)
  pastDueSince       DateTime?
  updatedAtStripe    DateTime                   // event.created — used to reject out-of-order events
  org                Org      @relation(fields: [orgId], references: [id], onDelete: Cascade)
}

enum SubStatus { TRIALING ACTIVE PAST_DUE CANCELED INCOMPLETE }
```

`stripeCustomerId` is stored the moment a customer is created, so a webhook arriving before the
checkout redirect still resolves to an org.

---

## Route handler

```ts
// src/app/api/webhooks/stripe/route.ts
import Stripe from "stripe";
import { headers } from "next/headers";
import { db } from "@/server/db";
import { env } from "@/env";

const stripe = new Stripe(env.STRIPE_SECRET_KEY);

export async function POST(req: Request) {
  // 1. RAW body. Never a parsed one — the signature is over the exact bytes.
  const raw = await req.text();
  const sig = (await headers()).get("stripe-signature");
  if (!sig) return new Response("Missing signature", { status: 400 });

  let event: Stripe.Event;
  try {
    event = stripe.webhooks.constructEvent(raw, sig, env.STRIPE_WEBHOOK_SECRET);
  } catch (err) {
    // Do not log the body; do not echo the error to the caller.
    console.error("stripe.webhook.signature_failed");
    return new Response("Invalid signature", { status: 400 });
  }

  // 2. Idempotency: unique insert wins the race, duplicates exit early.
  try {
    await db.stripeEvent.create({ data: { id: event.id, type: event.type } });
  } catch {
    return new Response("ok", { status: 200 });   // already handled
  }

  // 3. Handle. Return 2xx fast; push slow work (email, provisioning) to a queue.
  try {
    await handle(event);
  } catch (err) {
    // Delete the marker so Stripe's retry can succeed.
    await db.stripeEvent.delete({ where: { id: event.id } }).catch(() => {});
    console.error("stripe.webhook.handler_failed", { id: event.id, type: event.type });
    return new Response("Handler error", { status: 500 });   // 5xx ⇒ Stripe retries
  }

  return new Response("ok", { status: 200 });
}
```

Three things this gets right that hand-written handlers usually don't: the raw body, the idempotency
marker deleted on failure so the retry is not swallowed, and a 5xx (not a 200) when handling fails so
Stripe actually retries.

---

## Event table

| Event | Meaning | Action |
|---|---|---|
| `checkout.session.completed` | Payment succeeded, subscription created | Resolve org from `metadata.orgId`; upsert Subscription; grant entitlement |
| `customer.subscription.created` | Belt-and-braces for the above | Upsert; idempotent with the previous |
| `customer.subscription.updated` | Plan change, seat change, `cancel_at_period_end`, status change | Upsert status, priceId, seats, period end |
| `customer.subscription.deleted` | Subscription ended | Set `CANCELED`, revoke at `currentPeriodEnd` |
| `invoice.payment_failed` | Card declined | `PAST_DUE`, set `pastDueSince`, notify, start dunning |
| `invoice.payment_succeeded` | Renewal or recovery | Clear `pastDueSince`, extend `currentPeriodEnd` |
| `customer.subscription.trial_will_end` | 3 days left | Notify; prompt for a payment method |

**Missing `invoice.payment_failed` means users keep paid access for free indefinitely.** It is the
most commonly skipped handler and the most expensive.

---

## Ordering

Stripe does not guarantee order. An `updated` can arrive after the `deleted` it preceded.

```ts
const createdAt = new Date(event.created * 1000);

await db.subscription.updateMany({
  where: { stripeSubId: sub.id, updatedAtStripe: { lt: createdAt } },   // only if newer
  data: { status, priceId, currentPeriodEnd, updatedAtStripe: createdAt },
});
```

`updateMany` with the timestamp guard makes stale events no-ops instead of state corruption.

---

## Resolving the org

Set it at checkout creation, so every downstream event can find its way home:

```ts
await stripe.checkout.sessions.create({
  mode: "subscription",
  line_items: [{ price: priceId, quantity: seats }],   // priceId looked up server-side
  client_reference_id: orgId,
  metadata: { orgId },
  subscription_data: { metadata: { orgId } },          // so subscription.* events carry it too
  customer: existingCustomerId,                        // reuse; never create duplicates
  success_url: `${env.NEXT_PUBLIC_APP_URL}/billing?checkout=success`,
  cancel_url: `${env.NEXT_PUBLIC_APP_URL}/billing`,
});
```

`subscription_data.metadata` matters: without it, `customer.subscription.updated` has no `orgId` and
you are reverse-looking-up by customer id.

Never accept a price or amount from the client. Take a plan key, map it server-side to a price id.

---

## Local testing

```bash
stripe login
stripe listen --forward-to localhost:3000/api/webhooks/stripe   # prints the whsec_… for .env.local
stripe trigger checkout.session.completed
stripe trigger invoice.payment_failed
stripe trigger customer.subscription.deleted
```

Test card `4242…` succeeds; `4000 0000 0000 0341` attaches but fails on charge; `4000 0025 0000 3155`
requires 3DS. **Exercise the failure paths** — the happy path is the one that already works.

Then verify twice: send the same event twice and confirm the second is a no-op, and confirm a
tampered payload is rejected with 400.

---

## Before shipping

- [ ] Signature verified on the raw body; a tampered payload returns 400.
- [ ] Replaying an event is a no-op.
- [ ] Handler failure returns 5xx so Stripe retries, and the marker is removed.
- [ ] No network calls to Stripe on the request path — entitlements come from your DB.
- [ ] `invoice.payment_failed` handled, with a grace period and a user-visible message.
- [ ] Live-mode webhook endpoint registered with the live signing secret, separate from test.
- [ ] Two concurrent checkouts for one org cannot create two subscriptions.
