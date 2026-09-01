# Prisma patterns

Working shapes for the four things that go wrong most often.

---

## 1. Multi-tenant schema

Put `orgId` on **every** tenant-owned model, and index it as the first column of every composite
index. Do not rely on reaching the tenant through a relation — that makes every scoped query a join
and makes it easy to forget the scope entirely.

```prisma
model Org {
  id       String   @id @default(cuid())
  slug     String   @unique
  members  Member[]
  invoices Invoice[]
}

model Member {
  id     String   @id @default(cuid())
  orgId  String
  userId String
  role   Role     @default(MEMBER)
  org    Org      @relation(fields: [orgId], references: [id], onDelete: Cascade)
  user   User     @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([orgId, userId])          // one membership per user per org
  @@index([userId])                  // "which orgs am I in"
}

model Invoice {
  id         String        @id @default(cuid())
  orgId      String
  number     Int
  status     InvoiceStatus @default(DRAFT)
  totalCents Int                       // integer minor units, never Float
  currency   String        @db.Char(3)
  customerId String
  createdAt  DateTime      @default(now())
  org        Org           @relation(fields: [orgId], references: [id], onDelete: Cascade)

  @@unique([orgId, number])          // per-tenant invoice numbering
  @@index([orgId, status, createdAt(sort: Desc)])   // the list query's index
  @@index([orgId, customerId])
}

enum Role { OWNER ADMIN MEMBER }
enum InvoiceStatus { DRAFT ISSUED PAID VOID }
```

**Every index starts with `orgId`**, because every query filters on it. An index on `status` alone is
useless to a scoped query.

`onDelete: Cascade` on tenant-owned rows means deleting an org actually removes its data — which is
what a deletion request requires.

---

## 2. Transactions

Interactive transactions for multi-write invariants. **Pass `tx` down** — a helper that closes over
the global `db` silently escapes the transaction and you get a partial write with no error.

```ts
export async function issueInvoice(db: PrismaClient, orgId: string, invoiceId: string) {
  return db.$transaction(async (tx) => {
    const invoice = await tx.invoice.findFirstOrThrow({
      where: { id: invoiceId, orgId, status: "DRAFT" },
    });

    // per-tenant sequence, allocated inside the transaction
    const { _max } = await tx.invoice.aggregate({
      where: { orgId }, _max: { number: true },
    });

    const issued = await tx.invoice.update({
      where: { id: invoice.id },
      data: { status: "ISSUED", number: (_max.number ?? 0) + 1 },
    });

    await tx.activity.create({
      data: { orgId, type: "INVOICE_ISSUED", invoiceId: issued.id },
    });

    return issued;
  }, { timeout: 10_000 });
}
```

Rules:
- **No network calls inside a transaction** — no Stripe, no email, no webhook. They hold locks for as
  long as the third party takes and turn a slow API into a database outage. Enqueue them after commit.
- Keep transactions short; a long one blocks the rows it touches.
- The `@@unique([orgId, number])` constraint is the real guarantee against concurrent issuers — the
  aggregate above races, and the constraint is what catches it. Retry once on `P2002`.
- Prefer atomic operations (`{ increment: 1 }`) over read-modify-write.

---

## 3. Cursor pagination

Offset pagination gets slower as it goes (the database still walks the skipped rows) and skips or
repeats rows when data changes between pages. Use a cursor.

```ts
const listInput = z.object({
  limit: z.number().int().min(1).max(100).default(20),
  cursor: z.string().nullish(),
});

export const list = orgProcedure.input(listInput).query(async ({ ctx, input }) => {
  const rows = await ctx.db.invoice.findMany({
    where: { orgId: ctx.orgId },
    select: { id: true, number: true, status: true, totalCents: true, createdAt: true },
    orderBy: [{ createdAt: "desc" }, { id: "desc" }],   // tiebreaker keeps it deterministic
    take: input.limit + 1,                              // one extra = "is there more"
    ...(input.cursor ? { cursor: { id: input.cursor }, skip: 1 } : {}),
  });

  const nextCursor = rows.length > input.limit ? rows.pop()!.id : null;
  return { items: rows, nextCursor };
});
```

The **tiebreaker in `orderBy`** matters: without a unique second key, rows sharing a `createdAt` can
appear on two pages or on neither. The index must match the sort exactly, or Postgres sorts in memory.

Client side, `useInfiniteQuery` with `getNextPageParam: (last) => last.nextCursor`.

---

## 4. Soft delete

Only adopt it if you actually need restore or audit. It is not free: **every query must now exclude
deleted rows, and the one that forgets is a data leak.**

```prisma
model Document {
  id        String    @id @default(cuid())
  orgId     String
  deletedAt DateTime?

  @@index([orgId, deletedAt])
}
```

```ts
// Centralise the filter. Never hand-write `deletedAt: null` at call sites.
const active = (orgId: string) => ({ orgId, deletedAt: null });

await db.document.findMany({ where: { ...active(ctx.orgId), status: "PUBLISHED" } });
await db.document.updateMany({ where: active(ctx.orgId), data: { deletedAt: new Date() } });
```

- Unique constraints break under soft delete: a deleted row still occupies the unique slug. Either
  include `deletedAt` in the unique key or rename on delete.
- Have a real purge job; "soft deleted" is not "deleted" for a data-deletion request.
- A Prisma client extension can apply the filter globally — safer than remembering, but make it
  visible in the codebase so nobody is surprised by rows they cannot see.

---

## 5. Quick checks before merging a query

- [ ] Scoped by `orgId` **inside** the `where`, from the session.
- [ ] `select`/`include` explicit — no whole-row fetches.
- [ ] No `db` call inside a loop or `.map()`.
- [ ] An index exists that matches this filter **and** this sort.
- [ ] List endpoints are cursor-paginated and capped.
- [ ] Multi-write invariants are inside `$transaction`, with no network calls in it.
- [ ] Money is integer minor units.
- [ ] The migration was checked for table locks on a large table.
