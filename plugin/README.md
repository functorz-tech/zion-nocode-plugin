# zion-nocode — Claude Code、Codex、Cursor、QoderWork 与 Kimi 套件

自动生成 — 请勿手动编辑。技能内容由 `build_scripts/gen-skills.mjs` 生成，
使用 `node build_scripts/gen-skills.mjs` 重新生成。

`zion-platform` 技能会引导 Claude Code、Codex 或 Cursor 构建 Zion 项目，自动检测当前项目的
类型系统版本，并路由到相互隔离的对应能力目录。同一组合包提供五份宿主清单 — `.claude-plugin/plugin.json`（Claude Code）、
`.codex-plugin/plugin.json`（Codex）、`.cursor-plugin/plugin.json`（Cursor）、
`.qoder-plugin/plugin.json`（QoderWork）与 `.kimi-plugin/plugin.json`（Kimi） — 共享同一套 `skills/`、`hooks/` 与 `bin/`。Cursor 还会使用 `mcp.json` 中声明的 `zion` MCP 服务器；
Kimi 清单自带 `mcpServers` 与 `sessionStart` 声明；Claude Code 与 Codex 则忽略它们。其 CLI 配方直接通过 `npx -y zion-mcp@<version>` 调用锁定的已发布 CLI，
该版本锁定运行本套件的确切构建，因此即使 PATH 上全局安装了 `zion-mcp`（或 `zion`）也无法将其覆盖，
且不依赖宿主暴露 `PLUGIN_ROOT`/`CLAUDE_PLUGIN_ROOT`。随附的 `bin/zion-mcp` 启动器运行相同的
`npx -y zion-mcp@<version>`，可继续用于本地命令行。

## 本地使用

    # Claude Code
    claude --plugin-dir ./plugin
    /zion-nocode:zion-platform

    # Codex
    codex plugin marketplace add functorz-tech/zion-nocode-plugin
    codex plugin add zion-nocode@zion

    # Cursor（本地开发）— 建立符号链接，然后在 Cursor 的套件设置中启用
    ln -s "$PWD/plugin" ~/.cursor/plugins/local/zion-nocode

## 验证项目路由

在套件目录中单次加载项目，并检查返回的准确 `typeSystem`：

    ./bin/zion-mcp --no-daemon schema load \
      --args '{"projectExId":"<exId>"}' \
      --pretty

`schema load` 不直接接受 `--projectExId`。也可以先运行
`./bin/zion-mcp project set-current --projectExId <exId>` 固定项目，再运行
`./bin/zion-mcp schema load --pretty`。

套件始终运行**已发布**的 npm CLI，因此本地对 CLI 的改动不会通过它生效。
若要测试 CLI 本身，请直接运行（例如 `pnpm build && node bin/zion-mcp.js …`）。
