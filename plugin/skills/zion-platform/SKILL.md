---
name: zion-platform
description: >-
  Orientation and entry point for working on a Zion project. Load first for any Zion app task — designing or changing the data model, UI, server-side action flows, permissions, payments, or data bindings, or reading runtime logs — then follow it to the right capability. For Zion projects only; not generic programming help.
---

# Zion platform orientation

Before reading platform guidance, identify the active project and pin it when needed:

```bash
"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" projects search --projectName "My App"
"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" project set-current --projectExId <exId>
"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" schema load
```

Read the `typeSystem` field returned by `schema load` exactly, then route without asking the user:

- `pre_type_system_refactor` → read `pre/ROUTER.md`; never read `post/`.
- `post_type_system_refactor` → read `post/ROUTER.md`; never read `pre/`.

If schema loading fails, `typeSystem` is missing, or its value is unknown, stop and report the problem. Never guess a variant or ask the user to choose one.

Repeat this detection whenever the active project changes.

> **Always invoke the CLI exactly as written above** — via `${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp` — never bare `zion-mcp`, even though `--help` prints its own name that way. The `${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}` fallback resolves the plugin root under both Codex (`PLUGIN_ROOT`) and Claude Code (`CLAUDE_PLUGIN_ROOT`); a globally-installed `zion-mcp` (or `zion`) on `PATH` would otherwise shadow this plugin's pinned build.
