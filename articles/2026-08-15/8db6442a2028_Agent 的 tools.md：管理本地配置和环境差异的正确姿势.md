---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 33179
source: 综合讨论
publishedAt: 2026-08-15
---

## 背景

做 OpenClaw / Agent 本地自动化时，最常见的问题不是 Agent 不够聪明，而是同一个仓库在 A 机器能跑，换到 B 机器就挂：Chrome 路径不同、Python 版本不同、数据目录不同、API 代理不同、某些 CLI 工具根本没装。

这些差异通常散落在提示词、脚本、插件配置甚至聊天记录里。排查时只能反复问：“你本机路径是什么？”“这个命令为什么找不到？”Agent 也会因为环境信息不准确而反复试错，甚至误用旧路径执行破坏性操作。

## 问题

本地环境差异没有被结构化。Agent 每次都要从零开始猜测工具位置、可用性和参数格式。换机器、换系统、换插件时，没有一份可靠的“环境契约”告诉它哪些工具可用、命令怎么拼、失败了怎么降级。

## 做法：定义、渲染、校验

引入 `tools.md` 作为本地工具清单与环境差异的单一事实来源。核心不是把文件丢给 Agent，而是走三步：**定义模板 → 生成本地覆盖 → 渲染并校验**。

### 1. 定义模板

在仓库根目录建 `tools.example.md`，每个工具写清楚用途、命令模板、依赖环境变量、降级方案和启用状态：

```markdown
## browser
- purpose: open local page / export pdf
- command: "{{BROWSER_PATH}}" --headless --dump-dom "{{URL}}"
- env: BROWSER_PATH
- fallback: curl
- enabled: true

## ffmpeg
- purpose: video/audio transcode
- command: "{{FFMPEG_PATH}}" -i "{{INPUT}}" "{{OUTPUT}}"
- env: FFMPEG_PATH
- fallback: none
- enabled: true
```

关键点：命令里只写变量，不写具体路径。

### 2. 生成本地覆盖

复制为 `tools.local.md`，只写差异部分：

```markdown
BROWSER_PATH=/Applications/Google Chrome.app/Contents/MacOS/Google Chrome
FFMPEG_PATH=/opt/homebrew/bin/ffmpeg
```

`tools.local.md` 加入 `.gitignore`，个人路径和机器差异不要进版本库。

### 3. 渲染并校验

写一个启动前脚本 `scripts/render_tools.sh`，读取 `tools.example.md + tools.local.md + .env`，把 `{{VAR}}` 替换为实际值，输出 `tools.rendered.md`。渲染后做存在性检查：

- 用 `command -v` 或 `test -x` 检查命令/路径是否存在；
- 缺失则写入 warning，并把对应工具 `enabled` 改为 `false`，触发 fallback；
- 可选跑 `--version` 记录版本。

最后让 Agent 只读 `tools.rendered.md`。提示词固定为：“使用 tools.rendered.md 中的工具与命令，不要猜测路径。”这样 Agent 看到的永远是当前机器有效配置。

## 踩坑点

**敏感信息别写进 tools.md。** 即使 local 文件已 gitignore，仍可能被日志、截图或 Agent 输出泄露。密钥、token 一律用环境变量引用，不在文件中写明文。

**跨平台命令差异。** `rm -rf` 和 `Remove-Item`、`/` 和 `\` 不能混用。建议命令模板提供 per-platform 字段，如 `command_linux`、`command_macos`、`command_windows`，渲染时按系统选择。

**路径含空格。** `C:\Program Files\...` 直接拼字符串会出错。渲染层应使用 shell 数组或 JSON args，不要把命令拼成一个大字符串丢给 `subprocess`。

**CLI 输出版本变化。** 很多工具升级后输出格式变了，Agent 解析会失败。模板里加 `version_check` 和 `min_version`，启动时记录版本，必要时降级。

**fallback 不能缺。** 工具不可用时如果没有 fallback，Agent 会反复重试同一个坏工具。每个关键工具必须定义替代方案，或者干脆 `enabled: false` 让 Agent 跳过。

## 可复用建议

- **模板版本化**：`tools.example.md` 提交进 git，本地差异不进库。变更走 PR，附带环境差异说明。
- **脚本幂等**：每次启动都重新渲染，避免上次机器缓存影响。
- **生成 dry-run 报告**：渲染后给 Agent 一段简短报告，例如：“browser 可用；ffmpeg 缺失，已禁用；python 路径 /opt/homebrew/bin/python3”。这比让 Agent 自己试错可靠得多。
- **与 MCP/插件协同**：MCP server 的启动命令同样可以走 `tools.md` 的 command 字段，避免把启动参数硬编码在插件配置里。

## 总结

`tools.md` 不是给 Agent 读的说明书，而是本地环境的“编译目标”。把差异抽象成变量，把校验前置到启动阶段，只把渲染后的有效配置交给 Agent，能明显减少跨机器、跨系统的工具调用失败。

对 OpenClaw 用户来说，这套做法也适用于 MCP server、本地插件和自动化脚本。成本不高，但换来的是：换机器不慌、排障有据、Agent 不再瞎猜路径。

---

