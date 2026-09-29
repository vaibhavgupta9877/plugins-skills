# Gupta Technologies Claude Code Plugins

An installable Claude Code marketplace of engineering, security, product direction, critique, motion, web, Next.js, Flutter, and Rust skills. The marketplace contains nine plugins and 47 skills.

## Install

In Claude Code, add this marketplace and install the always-on plugins:

```text
/plugin marketplace add vaibhavgupta9877/plugins-skills
/plugin install core-eng@gupta-skills
/plugin install secure-ship@gupta-skills
/plugin install critique@gupta-skills
```

Install the stack-specific plugins that fit the project:

```text
/plugin install animate@gupta-skills
/plugin install web-ui@gupta-skills
/plugin install next-saas@gupta-skills
/plugin install flutter@gupta-skills
/plugin install rust-crate@gupta-skills
```

For a UI/experience design task needing a design-ready brand foundation, a distinctive award-grade design language, or a substantial surface redesign, optionally install:

```text
/plugin install product-direction@gupta-skills
```

Claude Code may ask you to confirm an installation scope. Run `/reload-plugins` if prompted to activate newly installed plugins.

## Plugins

| Plugin | Use it for |
|---|---|
| `core-eng` | Requirements, specs, code review, test strategy, debugging, and Git workflow; stack-agnostic. |
| `secure-ship` | Threat modeling, secrets and environment handling, dependency risk, and production-readiness gates. |
| `product-direction` | Design-ready brand context, a distinctive (non-generic) design language, and surface direction for product UI/experience work across stacks. |
| `critique` | Judgment-only reviews of UI, motion, accessibility, architecture, APIs, and copy. |
| `animate` | Motion decisions and implementation across web and Flutter. |
| `web-ui` | Web design systems, landing pages, UX copy, design engineering, and performance. |
| `next-saas` | Next.js T3 SaaS architecture, tRPC, Prisma, auth, billing, deployment, and observability. |
| `flutter` | Flutter architecture, UI, animation mechanics, desktop support, and release. |
| `rust-crate` | Rust crate architecture, API ergonomics, async services, FFI, publishing, and performance. |

## Repository guide

- `docs/00-INDEX.md` — ecosystem build specification, ownership boundaries, and verification requirements.
- `docs/01-core-eng.md` through `docs/09-product-direction.md` — per-plugin specifications.
- `plugins/<plugin>/skills/<skill>/SKILL.md` — skill instructions.
- `plugins/<plugin>/skills/<skill>/reference/` — optional on-demand references.

Validate an individual plugin locally with `claude plugin validate ./plugins/core-eng`, replacing `core-eng` with the plugin directory to check.
