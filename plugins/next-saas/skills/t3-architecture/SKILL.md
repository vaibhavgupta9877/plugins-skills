---
name: t3-architecture
description: Set up, structure, or restructure a T3 Next.js application — folder layout, server/client boundaries, env validation, layering conventions. Do NOT use for UI composition, that's web-ui/web-design-engineering.
---

# t3-architecture

## Owns
- Project scaffolding and directory conventions, layering, server/client boundary policy.
- Env schema wiring, the Prisma singleton, removing `create-t3-app` boilerplate.

## Does not own
- UI components and screens → `web-ui/*`.
- tRPC procedure and Prisma query design → `trpc-prisma-patterns`.
- Auth flows and billing → `auth-and-billing`.
- Deploy and CI → `deploy-observability`.
- Task plans and specs → `core-eng/spec-writer`.

**Stack:** Next.js App Router · TypeScript · tRPC · Prisma · Auth.js · Tailwind · Vercel-primary.
**Portability:** flag every Vercel-coupled decision and name the portable alternative —
see `deploy-observability` for the lock-in map.

---

## 1. Fresh setup — delete before you add

`create-t3-app` ships demonstration code. Remove it in the first commit, before it gets built on:

- `src/app/_components/post.tsx` and the example `post` router (`src/server/api/routers/post.ts`).
- The `Post` model in `schema.prisma` and its migration, if nothing depends on it yet.
- The Discord (or other placeholder) provider in the auth config, unless you are actually using it.
- Unused dependencies that came with those — uninstall rather than leaving them in `package.json`.
- The default landing page content.

**Then add these before the first feature**, in this order:

1. **Env schema** (`src/env.js`) — see section 4.
2. **`.env.example`**, committed, every key present, every value blank.
3. **Prisma singleton** (`src/server/db.ts`) — see section 5.
4. **Root error boundary and `not-found`**, both designed rather than default.
5. **A logger** — structured, not `console.log`. See `deploy-observability`.
6. **Base layout** with fonts (`next/font`), theme provider, and the token layer from
   `web-ui/design-system`.
7. **`CLAUDE.md`** documenting the conventions below, so they survive the next session.

---

## 2. Directory convention — feature-oriented, not type-oriented

```
src/
  app/                        # routing only: pages, layouts, route handlers
    (marketing)/
    (app)/dashboard/page.tsx
    api/webhooks/stripe/route.ts
  features/
    invoices/
      ui/                     # components for this feature
      server/                 # its tRPC router + service functions
      schema.ts               # its Zod schemas, shared client and server
      types.ts
  server/
    api/root.ts               # router composition
    api/trpc.ts               # context, middleware, procedure builders
    db.ts                     # Prisma singleton
    auth.ts
  components/ui/              # design-system primitives only, nothing feature-specific
  lib/                        # genuinely cross-cutting, each in a named module
  env.js
```

**Feature-oriented beats type-oriented.** `src/components` + `src/hooks` + `src/utils` means one
feature change touches four directories and no directory tells you what the app does. Grouping by
feature means a feature can be read, reviewed, and deleted as a unit.

`src/app` holds routing and nothing else — a page imports from `features/`, it does not implement.

**The "utils" trap: no `utils.ts` dumping ground.** Every helper lives next to the feature that uses
it, or in a named module (`lib/currency.ts`, `lib/dates.ts`). A file called `utils.ts` is where code
goes when nobody decided where it belongs, and it never gets cleaned up. The one conventional
exception is `lib/utils.ts` holding `cn()`, because shadcn expects it — keep it to that.

---

## 3. Server/client boundary policy

State it as a rule and enforce it mechanically:

- **Server components by default.** `"use client"` goes on the smallest leaf that needs
  interactivity, never on a page or layout.
- **Prisma is never importable from a client component.** Enforce it: put `import "server-only";` at
  the top of `src/server/db.ts` and every module under `src/server/**`. An accidental client import
  then fails the build instead of leaking your schema — or worse, your credentials — into the bundle.
- Mark genuinely client-only modules with `import "client-only";` for the same reason in reverse.
- Server data flows down as props. A client component that fetches what its server parent already
  had is a waterfall.
- Client components can accept server-rendered `children`, so an interactive wrapper does not force
  its whole subtree client-side. Use this instead of pushing the boundary upward.

In review, `"use client"` above a leaf is a finding, not a preference.

---

## 4. Env validation

`@t3-oss/env-nextjs` + Zod, with `server` and `client` split. **A missing or malformed variable must
fail the build, not production.**

- Constrain the shape, not just the type: `.url()`, `.startsWith("sk_")`, `.min(32)`.
- Never `.default()` a secret — a defaulted secret is a silent misconfiguration.
- Import the validated `env` object everywhere; `process.env` appears only inside `runtimeEnv`.
- `NEXT_PUBLIC_*` is public forever. Only public identifiers go in the `client` schema.

Full templates and the pre-commit secret scan: `secure-ship/secrets-config` and its
`reference/env-checklist.md`. Do not duplicate those rules here.

---

## 5. Prisma singleton

Next.js dev hot-reload re-evaluates modules, so a naive `new PrismaClient()` opens a new connection
pool on every save until the database refuses connections.

```ts
// src/server/db.ts
import "server-only";
import { PrismaClient } from "@prisma/client";
import { env } from "@/env";

const createClient = () =>
  new PrismaClient({ log: env.NODE_ENV === "development" ? ["query", "error", "warn"] : ["error"] });

const globalForPrisma = globalThis as unknown as { prisma?: ReturnType<typeof createClient> };

export const db = globalForPrisma.prisma ?? createClient();
if (env.NODE_ENV !== "production") globalForPrisma.prisma = db;
```

**Never instantiate per request.** In serverless, also configure pooling — see
`deploy-observability`; without it, connections exhaust under modest load.

Query logging on in dev is deliberate: it is how N+1 gets caught before production.

---

## 6. Layering inside a feature

```
ui/          → calls tRPC hooks, renders. No business rules.
server/      → procedures: authz, validation, orchestration.
  service    → business rules, testable without HTTP or React.
  db access  → Prisma calls.
schema.ts    → Zod, shared by both sides. One definition, two consumers.
```

The rule worth enforcing: **a procedure should read as a short list of steps.** When it grows
branching business logic, extract a service function that takes plain arguments and returns plain
data — then it is testable without a request context.

Share types through inference (`RouterOutputs`), not by hand-writing a second copy of the shape.

---

## 7. Restructuring an existing app

**Propose an incremental migration path, not a rewrite.** A rewrite stops feature work for weeks and
usually reproduces the same problems with newer names.

1. Establish the new convention and write it down.
2. Apply it to all **new** features immediately.
3. Move existing code **when you touch it for another reason**, one feature at a time.
4. Keep both structures working during the transition; do not leave the app half-migrated with
   imports pointing at moved files.

Sequence by pain: the feature you edit weekly moves first; the one untouched for a year may never
need to move.

**If a rewrite genuinely is cheaper, say so with the numbers** — how many files, how much of the
behaviour is covered by tests, what stops shipping meanwhile. Do not soften an honest estimate, and
do not recommend one because the codebase is unpleasant.
