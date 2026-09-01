---
name: copy-critique
description: Review interface copy — labels, buttons, errors, empty states, onboarding — for clarity and voice consistency. Do NOT use to write new copy, that's web-ui/ux-copy.
---

# copy-critique

> **Read first.** Invoke the `frontend-design` skill (plugin `frontend-design@claude-plugins-official`;
> on Linux sandboxes, `/mnt/skills/public/frontend-design/SKILL.md`) and read its **"More on writing in
> design"** section. That is the standard. Judge against it directly rather than restating it.

## Owns
- Judging existing interface text.

## Does not own
- Authoring copy → `web-ui/ux-copy`.
- Marketing and landing headlines → `web-ui/landing-page-craft`.
- Visual treatment of type → `ui-critique`.

**Findings only.**

---

## Output contract

Findings ranked `blocking` / `should-fix` / `nit`, each with **what · where · why it matters · the
rewritten line**. Always supply the replacement text — a copy finding without a rewrite is just a
complaint. End with the single most important change.

`blocking` is for copy that will cause a wrong action: a destructive confirmation whose button does
not name the destruction, an error that misdirects the fix, a label that means the opposite of what
the control does.

---

## The checks, highest value first

1. **Does the control name what happens?** "Save changes", not "Submit". "Delete 3 files", not "OK".
   A button is a promise about the next moment.
2. **Does the action keep its name through the whole flow?** The control says "Publish", the toast
   says "Published", the history entry says "Published". Drift between them is the most common real
   defect and the one users feel as "this app is confusing" without being able to say why.
3. **Do errors say what broke *and* how to fix it?** No apology, no vagueness, no "something went
   wrong". Name the thing and the next action. An error the user cannot act on is a `should-fix` at
   minimum.
4. **Are empty states an invitation or a dead end?** An empty screen is a moment for direction. "No
   invoices yet" is a dead end; "No invoices yet — create your first one" is an invitation.
5. **Is anything named after implementation rather than intent?** A person manages notifications, not
   webhook config. System vocabulary in the UI is a leak.
6. **Active voice, sentence case, plain verbs, no filler.** Being specific beats being clever.
7. **Does each element do exactly one job?** A label labels, an example demonstrates. Nothing quietly
   does double duty.

---

## Voice consistency

**Consistency across screens matters more than any individual line.** A perfect microcopy line in a
product that switches register between screens still reads as unfinished.

Sample at least five surfaces across different features — a form, an error, an empty state, a
confirmation, a settings row — and ask whether one person wrote them. Where they diverge, say which
one is the product's real voice and bring the others to it, rather than inventing a third.

Detect the voice from the product; do not impose one. If the product has no consistent voice yet, say
that as the finding and describe the voice the strongest existing screens imply.

---

## Report

Give a before/after table for anything you would change. Then the single most important change —
usually a vocabulary inconsistency running through a whole flow, not one bad line.
