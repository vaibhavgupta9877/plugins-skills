---
name: ux-copy
description: Write interface copy — button labels, error messages, empty states, onboarding, confirmations, form help text. Do NOT use to review existing copy, that's critique/copy-critique.
---

# ux-copy

> **Read first.** Invoke the `frontend-design` skill (plugin `frontend-design@claude-plugins-official`;
> on Linux sandboxes, `/mnt/skills/public/frontend-design/SKILL.md`) and read **"More on writing in
> design."** That is the standard. This skill supplies patterns and worked examples on top of it, never
> competing rules.

## Owns
- Authoring interface text.

## Does not own
- Reviewing existing copy → `critique/copy-critique`.
- Marketing and landing headlines → `landing-page-craft`.

---

## Before writing: detect the voice

**Read three or four existing screens and write down the product's actual voice** — register,
sentence length, whether it uses contractions, how it refers to the user. Then write in that voice.

Inventing a voice for one new screen is how a product ends up sounding like four different products.
If there is genuinely no consistent voice yet, say so, propose one in a sentence, and apply it
consistently from there.

---

## Patterns

### Buttons and CTAs
The control names what happens. The verb is the user's, not the system's.

| Instead of | Write |
|---|---|
| Submit | Save changes · Send invite · Create project |
| OK | Delete 3 files · Got it · Continue |
| Learn more | See how billing works |
| Yes / No | Discard draft / Keep editing |

### Confirmation dialogs
**The button says the action, never "OK".** The title asks the real question; the body says what is
irreversible.

> **Delete "Q3 forecast"?**
> This spreadsheet and its 42 revisions will be permanently deleted. This can't be undone.
> `[Cancel]` `[Delete spreadsheet]`

For genuinely destructive and high-volume actions, require typing the name. For everything else, an
undo toast beats a confirmation — it does not interrupt, and it is more forgiving.

### Errors
**What happened + how to fix it. No apology, never vague.**

| Instead of | Write |
|---|---|
| Something went wrong | Couldn't save — you're offline. We'll retry when you reconnect. |
| Invalid input | Enter a date in the future. |
| Error 403 | You don't have permission to edit this project. Ask an owner for access. |
| Upload failed | That file is 24MB. The limit is 10MB. |

Blame the situation, not the user. "Enter a valid email" beats "You entered an invalid email".
Put the message next to the field that caused it, not in a banner at the top.

### Empty states
**An invitation, not a dead end.** First-run empty and filtered-to-nothing empty are different.

> **No invoices yet**
> Invoices appear here once you send your first one.
> `[Create invoice]`

> **No results for "acme"**
> Try a different spelling, or clear the filters.
> `[Clear filters]`

### Loading
Name what is loading when it is slow enough to need words. "Loading…" says nothing; "Importing 1,240
contacts…" says the wait is proportional. Under a second, say nothing.

### Success toasts
**The past tense of the action's own verb.** "Publish" → "Published". Add the object when several
things could have happened: "Invoice sent to maria@acme.com". Include undo where undo exists.

### Form help and validation
- Help text before the field, describing the requirement — not after failure.
- Requirements stated up front ("At least 12 characters"), not revealed by rejection.
- Validate on blur, not on every keystroke; clear the error as soon as it is fixed.
- Label every field. Placeholder is not a label — it disappears exactly when it is needed.

### Permission requests
Ask in context, immediately before the value is delivered, and say what you will do with it.
"Allow notifications so we can alert you when a build fails" — never a cold system prompt on launch.

---

## Consistency rules

**An action keeps one name from control to toast to history entry.** "Publish" → "Published" →
"Published by you". Vocabulary drift across a flow is the most common real defect, and users
experience it as "this app is confusing" without being able to name why.

**Name things by what the user controls, never by system internals.** A person manages notifications,
not webhook config; they share a link, not generate an ACL entry.

Keep one term per concept across the whole product. Pick "project" or "workspace" and never alternate.

---

## Mechanics

- Sentence case for everything — buttons, labels, headings, menu items.
- Active voice, plain verbs, no filler. Specific beats clever.
- Numerals for numbers, including under ten, in UI.
- Second person for the user ("your projects"), first person only where the user speaks ("my account"
  as a label the user clicks). Do not mix both in one screen.
- No exclamation marks. No "Oops". No "Whoops". No emoji unless the product's brand already uses them.
- Each element does exactly one job — a label labels, an example demonstrates.
- Write for translation: avoid idiom, avoid concatenating sentence fragments in code, leave room for
  strings that get 30% longer in German.

## Sources
`frontend-design` (Anthropic), "More on writing in design."
