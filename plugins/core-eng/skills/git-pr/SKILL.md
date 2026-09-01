---
name: git-pr
description: Branch naming, commit messages, PR descriptions, and merge hygiene. Use when committing, opening a PR, or cleaning history. Do NOT use to review code content — that's code-review.
---

# git-pr

## Owns
- Commit, branch and PR conventions; changelog entries; history hygiene.

## Does not own
- Review of the code itself → `code-review`.
- Release tagging and publishing for crates → `rust-crate/publish-open-source`.

---

## Detect before you impose

Run `git log --oneline -30` and read the last few PR titles **before** writing anything.
Consistency with the repo beats any external standard. If the repo writes
`ORD-1123 fix invoice rounding`, write that — do not convert it to Conventional Commits
because you prefer them.

Only propose a convention change when the user asks, or when the log shows no convention
at all.

---

## Default, when no convention exists

**Conventional Commits**, imperative mood:

```
feat: add discount rows to invoice totals

Totals were computed before discounts loaded, so discounted orders were
charged the full amount. Computation now happens inside buildInvoice where
both are in scope.

Closes #412
```

- Types: `feat` · `fix` · `chore` · `refactor` · `docs` · `test` · `perf` · `build` · `ci`.
- Subject: imperative, lower case, no trailing period, under ~72 characters.
- **Body explains *why*, not *what*.** The diff already says what changed.
- Breaking changes: `feat!:` and a `BREAKING CHANGE:` footer stating the migration.

**Branches:** `<type>/<short-kebab-description>` — `fix/invoice-discount-rounding`.

---

## Commit granularity

One logical change per commit; a commit described with "and" is two commits. Keep refactors
and behaviour changes in separate commits — a reviewer cannot see a bug hidden inside a
400-line rename. Never mix in unrelated formatting churn.

---

## PR description template

```markdown
## What
One or two sentences.

## Why
The problem, and why this approach over the alternatives.

## How to test
Exact steps or commands a reviewer can run.

## Risk
What could break, and the rollback.

## Screenshots
Before / after — required for any UI change.
```

Link the issue. If the PR contains a decision the user has not seen, put it at the top,
not in a code comment.

---

## History hygiene

- **Never force-push a shared branch.** Never rewrite history that has been pulled.
- Rebasing your own unpushed branch onto the base is fine and preferred over a merge commit.
- Squash-merge when the branch is a stream of "wip" and "fix review"; preserve history when
  the commits were curated and each one stands alone.
- Do not commit generated files, lockfile churn from a different package manager, `.env`,
  credentials, or editor config, unless the repo already tracks them.
- `--no-verify` and skipping signing are for the user to ask for, never for you to reach for.
  A failing hook is a finding, not an obstacle.

---

## Changelog

If the repo has a `CHANGELOG.md`, update it in the same PR. Entries describe user-visible
change, in the user's vocabulary — not the internal function that changed.

**Public Rust crates:** commits and changelog entries are read by strangers who will never
see the code. Write them for an external audience: what changed in the public API, what
breaks, what to do about it. See `rust-crate/publish-open-source` for release mechanics.
