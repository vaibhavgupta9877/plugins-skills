---
name: prod-readiness-gate
description: Run the pre-release checklist for a Flutter app, Rust crate, or Next.js SaaS before shipping. Use before any production deploy, store submission, or crate publish. Do NOT use during feature development.
---

# prod-readiness-gate

## Owns
- Go/no-go release gating. Output is PASS or BLOCK.
- The per-stack ship checklist and the evidence for each item.

## Does not own
- How to build/sign/upload the artifact → `flutter/flutter-release`,
  `rust-crate/publish-open-source`, `next-saas/deploy-observability`.
- Diff review → `core-eng/code-review`. Attack surface → `threat-model`.
- Dependency findings → `dependency-audit` (this gate consumes its result).

---

## This is a gate, not advice

Output is **PASS** or **BLOCK**, with the specific failing items and what each would cost
in production.

**Do not soften a BLOCK.** No "consider", no "you may want to", no PASS-with-caveats. If
one universal item fails, the answer is BLOCK plus the shortest path to green. The user
can override — that is their call, made explicitly, not one you make for them by hedging.

A checked item needs **evidence**, not an assumption. "Error tracking is wired" means you
saw the config *and* a test event arrived. Anything you could not verify is reported as
**UNVERIFIED**, listed separately, and never counted as a pass.

---

## Step 1 — Identify the stack and load its checklist

Detect from the repo: `pubspec.yaml` → Flutter · `Cargo.toml` with `[package]` and a
publish target → Rust crate · `next.config.*` → Next.js SaaS.

Then read **only** the relevant file:
- `reference/checklists/nextjs.md`
- `reference/checklists/flutter.md`
- `reference/checklists/rust.md`

A monorepo shipping two things gets two gates, run separately.

---

## Step 2 — Universal items (all stacks, no exceptions)

- **No secrets in the shipped bundle.** Grep the build output, not the source. For Flutter,
  the binary; for Next.js, the client chunks; for Rust, `cargo package --list`.
- **Error tracking wired and verified with a real test event.** "Sentry is installed" is
  not a pass — an event must have arrived in the dashboard from a release build.
- **No `console.log` / `print` / `dbg!` / `debugPrint` in shipped code paths.** Especially
  any that log request bodies, tokens, or PII.
- **Dependency audit clean of criticals** — run `dependency-audit` and consume its verdict.
- **A rollback plan exists and you can describe it in two sentences.** "Redeploy the
  previous Vercel deployment" is a plan. "We'd figure it out" is a BLOCK.
- **Migrations are reversible, or explicitly one-way with a written reason.** A destructive
  migration deployed with no down path and no backup verified is an automatic BLOCK.
- **The build being gated is the build that ships.** Release mode, production env, real
  config — not a debug build with the prod flag flipped.

---

## Step 3 — Run the stack checklist

Work the loaded file item by item. For each: PASS with evidence, BLOCK with the failure,
or UNVERIFIED with what you would need.

The three failure modes that account for most real incidents in this stack set:

- **Next.js SaaS — IDOR.** Every tRPC procedure touching tenant data must scope by tenant
  in the query itself. This is the default failure. Check every procedure, not a sample.
- **Flutter — the device you did not test on.** A release build on a real low-end Android
  device, and clean-machine launches for Windows and Linux. Emulators hide startup crashes,
  ABI mistakes, missing runtime libraries, and jank.
- **Rust crate — the publish is irreversible.** `cargo publish` cannot be undone, only
  yanked. `--dry-run` clean, docs build, doctests pass, semver reviewed, MSRV declared.

---

## Step 4 — Verdict

```
VERDICT: BLOCK

Blocking (2)
1. tRPC `invoice.get` looks up by id with no orgId filter — src/server/api/routers/invoice.ts:41
   Cost: any authenticated user can read any tenant's invoices. This is the breach.
   Fix: add `orgId: ctx.session.orgId` to the where clause; add a cross-tenant test.
2. No verified error-tracking event from a production build.
   Cost: the first production failure is invisible until a user reports it.

Unverified (1)
- DB restore has never been exercised. Cannot confirm backups are usable.

Highest risk if you ship anyway
The IDOR. A single crafted request enumerates every tenant's invoices — a disclosure
breach with notification obligations, not a bug you patch quietly on Monday.
```

Always end with **the single highest-risk item and what it would cost if it fails.** One
paragraph, concrete, in business terms. That sentence is what makes the gate land.

On a genuine PASS, say PASS plainly and stop. Manufacturing a blocker to look thorough
makes the next gate worthless.
