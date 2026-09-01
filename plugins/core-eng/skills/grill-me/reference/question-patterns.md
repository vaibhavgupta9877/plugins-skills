# Question patterns by work type

Reusable question sets. Pick the matching block, cut what the repo already answers,
keep at most 3 per round. Options shown are starting points — always rewrite them in
the user's own domain vocabulary.

---

## New feature

1. **What does success look like the day after ship?**
   - A specific user can complete a specific flow end to end
   - An internal team stops doing something manually
   - A metric moves (name it)
   - It exists behind a flag, unshipped, for later

2. **Who is this for, first?**
   - Existing users, all of them
   - A single design-partner / internal team
   - New signups only
   - Ourselves (internal tooling)

3. **What is explicitly out of scope for this pass?**
   (multi-select: migration of existing data · admin UI · i18n · mobile layout ·
   offline · permissions/roles · analytics · rate limiting)

4. **Where does the data live?**
   - Existing table, new columns
   - New table in the same DB
   - New external service
   - Derived at read time, stored nowhere

---

## Refactor

1. **What is the actual pain?** — a bug class that keeps recurring · onboarding cost ·
   blocked feature · performance · aesthetic discomfort *(if it is the last one, say so
   and recommend not doing it)*
2. **Behaviour change allowed?** — strictly none, tests must pass unchanged · minor
   observable changes fine · this is a rewrite with new behaviour
3. **Blast radius ceiling** — one module · one layer across modules · whole package · no limit
4. **Landing strategy** — one big PR · incremental with both paths live · branch-and-cut-over

---

## New project

1. **What kills this project if it is wrong on day one?** — data model · auth/tenancy ·
   hosting/deploy target · the framework choice
2. **Timeline shape** — demo in days · production in weeks · long-lived, correctness first
3. **Who maintains it in six months?** — you alone · a team · unknown/handover · nobody, throwaway
4. **Deploy target** — locked already (name it) · open, recommend one · must be self-hostable

---

## Bug

*Prefer `debug` unless the fix implies a design change. If it does, ask:*

1. **Is the reported symptom the bug, or a consequence?**
2. **Fix scope** — patch the symptom now, ticket the cause · fix the cause, accept a
   bigger diff · both, in two commits
3. **Does this need a regression test that fails today?** (default: yes)

---

## Integration (third-party service)

1. **Is the vendor decided, or are we choosing?**
2. **Failure policy** — degrade silently · fail the user's request · queue and retry · block deploy
3. **Where do credentials live?** — existing secret store · env vars · needs a decision
4. **Sandbox available for tests, or do we mock the contract?**

---

## Redesign (UI)

1. **What specifically is wrong now?** — hierarchy/attention · density · brand/aesthetic ·
   flow has too many steps · accessibility
2. **Constraint set** — must keep current design tokens · tokens may change, components
   may not · everything is open
3. **Scope** — one screen · one flow · the whole product surface

*After locking, hand UI work to `critique/*` for judgement and `web-ui` / `flutter` /
`animate` for execution.*

---

## Question smells — never ask these

- "Should I use best practices?" — always yes, not a question.
- "Do you want tests?" — assume yes; ask about *layer*, not existence.
- "What framework are you using?" — read the manifest.
- Anything with an obvious answer; it burns one of your three slots.
- Two options that are the same decision worded differently.
