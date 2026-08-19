---
name: zion-platform
description: >-
  Orientation and entry point for working on a Zion project. Load first for any Zion app task — designing or changing the data model, UI, server-side action flows, permissions, payments, or data bindings, or reading runtime logs — then follow it to the right capability. For Zion projects only; not generic programming help.
---

# Zion platform orientation

Before reading platform guidance, identify the active project and pin it when needed:

```bash
npx -y zion-mcp@2.6.2 projects search --projectName "My App"
npx -y zion-mcp@2.6.2 project set-current --projectExId <exId>
npx -y zion-mcp@2.6.2 schema load
```

Read the `typeSystem` field returned by `schema load` exactly, then route without asking the user:

- `pre_type_system_refactor` → read `pre/ROUTER.md`; never read `post/`.
- `post_type_system_refactor` → read `post/ROUTER.md`; never read `pre/`.

If schema loading fails, `typeSystem` is missing, or its value is unknown, stop and report the problem. Never guess a variant or ask the user to choose one.

Repeat this detection whenever the active project changes.

> **Always invoke the CLI exactly as written above** — via `npx -y zion-mcp@2.6.2` — never bare `zion-mcp`, even though `--help` prints its own name that way. The `@2.6.2` version pin runs this plugin's exact published build, so a globally-installed `zion-mcp` (or `zion`) on `PATH` can't shadow it.
