---
name: trpc-prisma-patterns
description: Write tRPC routers and Prisma queries — procedure design, input validation, authorization middleware, transactions, N+1 avoidance, migrations. Do NOT use for public API surface review, that's critique/api-critique.
---

# trpc-prisma-patterns

## Owns
- Procedure implementation, Zod inputs, authorization middleware.
- Prisma schema, queries, transactions, migrations, indexes.
- Client-side cache invalidation policy for tRPC.

## Does not own
- Auth session setup and billing → `auth-and-billing`.
- Designing a new surface before implementation → `api-design`.
- Reviewing an existing surface → `critique/api-critique`.
- Deploy-time DB configuration and pooling → `deploy-observability`.

---

## 1. THE RULE: tenant scoping

**Every query touching tenant data must filter by a scope taken from the session, never from client
input.** Accepting an `orgId` from the client is the IDOR bug that breaks real SaaS products, and it
is the single most common serious defect in this stack.

```ts
// WRONG — the client chooses whose data to read
.input(z.object({ orgId: z.string(), invoiceId: z.string() }))
.query(({ ctx, input }) => ctx.db.invoice.findUnique({ where: { id: input.invoiceId } }))

// RIGHT — the scope comes from the session, and it is in the query, not a check after it
.input(z.object({ invoiceId: z.string() }))
.query(({ ctx, input }) =>
  ctx.db.invoice.findFirstOrThrow({
    where: { id: input.invoiceId, orgId: ctx.orgId },   // both, in one statement
  }))
```

Fetch-then-compare is not equivalent: it leaks existence, it is easy to forget, and one early
`return` skips it. Put the scope in the `where`.

**Enforce it with middleware, not with discipline.** Authorization is infrastructure:

```ts
// src/server/api/trpc.ts
export const protectedProcedure = t.procedure.use(({ ctx, next }) => {
  if (!ctx.session?.user) throw new TRPCError({ code: "UNAUTHORIZED" });
  return next({ ctx: { ...ctx, session: { ...ctx.session, user: ctx.session.user } } });
});

export const orgProcedure = protectedProcedure.use(async ({ ctx, next }) => {
  const orgId = ctx.session.user.activeOrgId;
  if (!orgId) throw new TRPCError({ code: "FORBIDDEN" });
  return next({ ctx: { ...ctx, orgId } });   // injected, non-optional, from here down
});
```

Then `orgId` is always in context and never in an input. Write one test that asserts a cross-tenant
read returns `NOT_FOUND`, and keep it green forever.

---

## 2. Procedures model use-cases, not tables

**A router that mirrors your Prisma models one-to-one is a design smell.** It pushes orchestration
into the client, where it gets implemented twice, differently, and without a transaction.

`createInvoiceAndNotify` beats `invoice.create` + `email.send` + `activity.log` from the client.
Name procedures after what the user is doing: `invoice.issue`, `member.invite`, `subscription.cancel`.

Nest at most two levels: `invoice.list`, `invoice.issue`, `org.member.invite`.

---

## 3. Inputs

**Every procedure has a Zod schema that accepts exactly what it uses.** No pass-through objects, no
`z.any()`, no fields the handler ignores.

- Constrain: `z.string().uuid()`, `z.string().email()`, `.min()`/`.max()` on anything free-text,
  `.positive().int()` on counts, enums for enums.
- Cap list inputs (`z.array(...).max(100)`) — an unbounded array is a free denial-of-service.
- Define the schema in `features/<feature>/schema.ts` and import it on both sides, so the form and
  the procedure cannot drift.
- Money is integer minor units, never a float.

---

## 4. Errors

`TRPCError` with the correct code — the code drives client behaviour, so getting it wrong sends users
to the wrong place:

`UNAUTHORIZED` (not signed in) · `FORBIDDEN` (signed in, not allowed) · `NOT_FOUND` ·
`CONFLICT` (duplicate, version clash) · `BAD_REQUEST` (validation) · `TOO_MANY_REQUESTS` ·
`PRECONDITION_FAILED` (plan limit, unverified email).

**Never leak internal messages or stack traces to the client.** Log the detail server-side with the
request id; return the safe message. Configure `errorFormatter` to strip anything unexpected in
production, and treat an unhandled Prisma error reaching the client as a defect.

For a cross-tenant miss, return `NOT_FOUND` rather than `FORBIDDEN` — `FORBIDDEN` confirms the record
exists.

---

## 5. Prisma correctness

- **`select` or `include` explicitly. Never fetch whole rows by default** — it ships columns you
  didn't mean to expose (password hashes, internal flags) and costs bandwidth on every call.
- `findUniqueOrThrow` / `findFirstOrThrow` where absence is exceptional; the `null` branch you skip
  is the bug you ship.
- **`$transaction` for multi-write invariants** — two writes that must both land or neither. Pass the
  transactional client through; a nested call using the global `db` silently escapes the transaction.
- `Promise.all` for genuinely independent reads.
- Use `updateMany` with the tenant scope for bulk operations, and check the returned count.
- Unique constraints in the schema, not just validation in code. Concurrent requests defeat
  check-then-write every time.

**N+1 is the default failure.** A `db` call inside a `.map()` over rows is the shape.

```ts
// N+1
const invoices = await db.invoice.findMany({ where: { orgId } });
for (const i of invoices) i.customer = await db.customer.findUnique({ where: { id: i.customerId } });

// One query
const invoices = await db.invoice.findMany({
  where: { orgId },
  include: { customer: { select: { id: true, name: true } } },
});
```

**Enable query logging in dev and actually read it.** Load a page, count the queries. That habit
catches N+1 before it reaches a production dataset where it becomes a timeout.

---

## 6. Migrations

- `prisma migrate dev` locally; **`prisma migrate deploy` in CI**. Never `db push` against production
  — it can drop columns without warning and leaves no migration history.
- Review every migration for **lock duration on large tables** before it ships. On Postgres:
  - Adding a nullable column with no default is fast.
  - **Adding a NOT NULL column with a default rewrites the table on older Postgres** and locks it.
    Add nullable → backfill in batches → set NOT NULL.
  - Adding an index locks writes unless you use `CREATE INDEX CONCURRENTLY` (which Prisma cannot
    generate — write it in a manual migration).
  - Renaming or dropping a column breaks the currently deployed code the moment it runs.
- **Expand → migrate → contract** for anything breaking: add the new column, deploy code writing
  both, backfill, deploy code reading the new one, then drop the old in a later release. Each step is
  independently reversible.
- **Add the index in the same PR as the query that needs it.** An index added "later" is added never,
  and the query that needed it is now a production incident.
- Cursor pagination over offset — offset gets slower as it goes and skips rows when data changes
  under it.

---

## 7. Client cache and invalidation

**State the invalidation policy, or stale UI ships.** After a mutation, invalidate every query whose
result it changed — not just the obvious one:

```ts
const utils = api.useUtils();
const issue = api.invoice.issue.useMutation({
  onSuccess: async () => {
    await Promise.all([utils.invoice.list.invalidate(), utils.org.usage.invalidate()]);
  },
});
```

Optimistic updates for mutations that almost always succeed: `cancel()` the query, snapshot, apply,
roll back **visibly** in `onError`, and settle in `onSettled`. A silent rollback is worse than a
spinner.

Return the updated entity from a mutation — it saves the client a round-trip and makes the optimistic
reconcile exact.

See `reference/prisma-patterns.md` for transaction, cursor pagination, soft-delete, and multi-tenant
schema patterns.
