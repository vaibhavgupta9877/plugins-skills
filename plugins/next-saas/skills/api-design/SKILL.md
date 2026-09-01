---
name: api-design
description: Design the shape of new API surfaces — tRPC router structure, REST route handlers, webhooks, error contracts, versioning. Do NOT use to review an existing API, that's critique/api-critique.
---

# api-design

## Owns
- Designing a new surface before it is implemented.

## Does not own
- Implementation → `trpc-prisma-patterns`.
- Reviewing an existing surface → `critique/api-critique`.
- Auth and entitlement design → `auth-and-billing`.

---

## 1. Design from the call site backwards

**Write the call site first**, as the consumer would want it to read, then build the procedure that
makes that call site true.

```ts
// Write this first:
const { data } = api.invoice.list.useQuery({ status: "OVERDUE" });
await api.invoice.issue.useMutation().mutateAsync({ invoiceId });
```

If the call site needs three round-trips and a client-side join to do one user-visible thing, the
surface is wrong. Design from **use-cases**, not from tables: a router that mirrors the Prisma schema
one-to-one has pushed the real design work onto every consumer.

---

## 2. Router structure

- Mirror **product domains**, not database tables: `invoice`, `org`, `billing`, `member`.
- **Nest at most two levels.** `org.member.invite` is fine; a fourth level means the domain wants
  splitting.
- Name procedures after the action the user takes: `issue`, `void`, `invite`, `cancel` — not
  `updateStatus`.
- Keep read and write names symmetric across the surface: if it is `list`/`byId`/`create` in one
  router, it is that everywhere. Guessability is the whole value of consistency.
- Compose routers per feature (`features/invoices/server/router.ts`) and assemble in
  `server/api/root.ts`.

---

## 3. tRPC or a REST route handler

Default to tRPC. Use a **route handler** for exactly these:

- **Webhooks and third-party callbacks** — the caller cannot speak tRPC, and you need the raw body.
- **File upload and download** — streaming, `Content-Disposition`, signed URLs.
- **Anything consumed by a non-TypeScript client** — a mobile app, a partner, a script, a CLI.
- **Health checks and OG image generation.**

Everything else is tRPC, because the end-to-end type inference is the reason the stack was chosen.

The moment a surface has an external consumer, it stops being an internal API and inherits every rule
in section 6.

---

## 4. One error contract

**One shape, typed, documented, used everywhere.** Consumers write error handling once.

For tRPC, that is `TRPCError` with a code from the fixed set: `UNAUTHORIZED`, `FORBIDDEN`,
`NOT_FOUND`, `CONFLICT`, `BAD_REQUEST`, `TOO_MANY_REQUESTS`, `PRECONDITION_FAILED`. Decide *now*
which domain failures map to which code, and write it down — an inconsistent mapping sends users to
the wrong recovery path.

For REST, one JSON body shape for every error: a stable machine `code`, a human `message`, and an
optional `details` for field errors. Never a bare string, never a stack trace, never a different shape
per endpoint.

Add a machine-readable code for anything the client must branch on (`PLAN_LIMIT_REACHED`,
`SEAT_LIMIT_REACHED`) — branching on message text breaks the first time you improve the wording.

---

## 5. Shape decisions to make once

**Pagination: cursor-based by default.** Offset pagination gets slower as it goes and silently skips
or repeats rows when data changes underneath. Fix the response shape now —
`{ items, nextCursor }` — and use it for every list on the surface.

**Idempotency: any mutation that could be retried needs a key.** Name them explicitly at design time:
anything that charges money, sends a message, provisions a resource, or is triggered by a webhook.
Accept a client-supplied `idempotencyKey`, store it with the result, and return the stored result on
replay. A retry after a timeout is normal, not exceptional.

**Filtering and sorting:** an allow-list of sortable fields, never a raw column name from the client.
Every filter needs a matching index — decide both together.

**Partial responses:** decide once whether the surface supports field selection. Usually the answer
for tRPC is no, and the shape is fixed per procedure. Say so rather than growing it accidentally.

**Bulk operations:** design them up front for anything a user will do to many rows. Retrofitting
bulk means N round-trips in production first.

---

## 6. Versioning

- **Internal tRPC: no versioning.** The client and server ship together; change both in one commit.
  Adding version prefixes to an internal API is cost with no benefit.
- **Anything external: version from day one.** `/api/v1/…`. Retrofitting versioning once third
  parties depend on you is expensive and slow, and you cannot do it unilaterally.
- Additive changes are safe; removing a field, tightening validation, or changing an error code is
  breaking — including making a previously optional Zod input required.
- For external surfaces, decide the deprecation policy at design time: how long an old version lives,
  how consumers are told. Route the surface through `critique/api-critique` before it goes public.

---

## 7. Long-running work does not belong in a request

Name the approach explicitly at design time — this is where "we'll figure it out" costs the most.

**The Vercel constraint is real:** serverless functions have an execution timeout (seconds to a
couple of minutes depending on plan and runtime). Report generation, bulk import, video processing,
and large exports will exceed it, and the failure mode is a truncated response with no error you can
catch.

The pattern: accept the request, validate it, persist a job row, return `202` with a job id, process
out-of-band, and let the client poll or subscribe. Design the job status shape at the same time.

For the runner, name the choice and its portability cost:
- **Vercel Cron + a queue table in Postgres** — simplest, portable, fine at low volume.
- **QStash / Inngest / Trigger.dev** — real queueing and retries, a third-party dependency.
- **A separate always-on worker** (Fly, Railway, a container) — most portable, most operational work.

Whatever you choose, say what moving off Vercel would cost. See `deploy-observability` for the
full lock-in map.
