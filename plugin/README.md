# zion-nocode — Claude Code、Codex 与 Cursor 套件

自动生成 — 请勿手动编辑。技能内容由 `build_scripts/gen-skills.mjs` 生成，
使用 `node build_scripts/gen-skills.mjs` 重新生成。

`zion-platform` 技能会引导 Claude Code、Codex 或 Cursor 构建 Zion 项目（类型系统重构前的组合包），
并路由到各能力子文档。同一组合包提供三份宿主清单 — `.claude-plugin/plugin.json`（Claude Code）、
`.codex-plugin/plugin.json`（Codex）与 `.cursor-plugin/plugin.json`（Cursor） — 共享同一套
`skills/`、`hooks/` 与 `bin/`。Cursor 还会使用 `mcp.json` 中声明的 `zion` MCP 服务器；
Claude Code 与 Codex 则忽略它，改为通过 bin 启动器调用 CLI。其 CLI 配方以绝对路径调用启动器
（`${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp`，在两种宿主下均可解析），而非裸命令，
因此即使 PATH 上全局安装了 `zion-mcp`（或 `zion`）也无法将其覆盖。启动器始终通过
`npx -y zion-mcp@<version>` 运行锁定的已发布 CLI，因此无论从套件市场安装还是本地运行，套件行为都完全一致。

## 本地使用

    # Claude Code
    claude --plugin-dir ./plugin
    /zion-nocode:zion-platform

    # Codex
    codex plugin marketplace add functorz-tech/zion-nocode-plugin
    codex plugin add zion-nocode@zion

    # Cursor（本地开发）— 建立符号链接，然后在 Cursor 的套件设置中启用
    ln -s "$PWD/plugin" ~/.cursor/plugins/local/zion-nocode

套件始终运行**已发布**的 npm CLI，因此本地对 CLI 的改动不会通过它生效。
若要测试 CLI 本身，请直接运行（例如 `pnpm build && node bin/zion-mcp.js …`）。
