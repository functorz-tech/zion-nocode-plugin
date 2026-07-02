# Zion no-code plugin

Build and modify [Zion](https://functorz.com) no-code apps from your AI coding assistant — the data
model, UI components, server-side action flows, data bindings, payments, permissions, and runtime
logs — powered by the `zion-mcp` CLI plus guided skills. For Zion projects only.

## Install

### Claude Code

From your terminal:

```bash
claude plugin marketplace add functorz-tech/zion-nocode-plugin
claude plugin install zion-nocode@zion
```

Or inside Claude Code:

```
/plugin marketplace add functorz-tech/zion-nocode-plugin
/plugin install zion-nocode@zion
/reload-plugins
```

### Codex

```bash
codex plugin marketplace add functorz-tech/zion-nocode-plugin
codex plugin add zion-nocode@zion
codex plugin list | grep zion
```

### Cursor

Cursor 2.5+ reads this repo as a plugin marketplace (it ships `.cursor-plugin/marketplace.json`),
installing both the `zion-platform` skill **and** the `zion` MCP server (`mcp.json`):

- **Team / org (Cursor Teams or Enterprise):** an admin adds this Git repo as a team marketplace;
  members then install `zion-nocode` from Cursor's plugin list.
- **Local / development:** symlink the plugin into Cursor's local plugins directory, then enable it
  in Cursor's plugin settings:

```bash
ln -s "$PWD/plugin" ~/.cursor/plugins/local/zion-nocode
```

A public Cursor Marketplace listing is submitted at cursor.com/marketplace/publish (open source,
manually reviewed).

### Other MCP-compatible tools (Windsurf, Cline, Claude Desktop, Qoder, Trae, …)

These don't read the plugin marketplace, but they can use the underlying `zion-mcp` MCP server
directly. Add it to your tool's MCP config:

```json
{
  "mcpServers": {
    "zion": {
      "command": "npx",
      "args": ["-y", "zion-mcp@latest", "mcp"]
    }
  }
}
```

You get Zion's tools (schema, action flows, data bindings, logs, …); the guided `zion-platform`
skill is plugin-only (Claude Code, Codex, Cursor).

## First use

The plugin runs `zion-mcp` through `npx`, so **Node.js 18+** is required. On first use you'll be
asked to authenticate; you can also do it ahead of time:

```bash
npx -y zion-mcp login
```

Then ask your assistant to work on your Zion app — it loads the `zion-platform` skill and takes
it from there.
