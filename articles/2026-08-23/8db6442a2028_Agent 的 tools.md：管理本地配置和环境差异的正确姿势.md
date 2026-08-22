---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 34262
source: 综合讨论
publishedAt: 2026-08-23
---

# Agent 的 tools.md：管理本地配置和环境差异的正确姿势

## 背景

在 OpenClaw 系 Agent 项目里，`tools.md` 经常被当作“工具清单 + 调用说明”。但真正踩坑的地方不是怎么描述工具，而是怎么让同一份 `tools.md` 在不同机器、不同 shell、不同 Node/Python 版本之间保持可用。一个仓库在 macOS 上跑得好好的，换到 Linux 或 Windows 就可能出现工具找不到、MCP server 起不来、脚本权限不够、路径解析错误。

问题通常不是 Agent 不够聪明，而是我们把太多“环境相关事实”塞进了 `tools.md`，却没有分层。

## 常见反模式

- 把绝对路径写进 `tools.md`：`/Users/alice/.nvm/versions/node/v20/bin/node`，换机器必挂。
- 密钥、Token 直接写进 `tools.md` 并提交到 git。
- 写 bash 命令时默认 GNU 工具行为，但 macOS 是 BSD 实现，例如 `sed -i`、`grep -P`。
- MCP server 的启动命令写死 `npx -y xxx`，但本地 Node 版本不一致、`npx` 路径不对。
- 配置一次性生成，从不校验，Agent 无法判断当前环境是否就绪。

## 正确做法：分层、探测、校验、生成

### 1. 分层：静态声明与动态环境分离

`tools.md` 只保留工具语义和稳定接口，环境差异下沉到 `tools.local.md`（gitignore）或 `.env`。

```yaml
# tools.md —— 只写接口
tool: screenshot
command: "{{PYTHON_BIN}}"
args: ["scripts/screenshot.py", "--output", "{{OUTPUT_DIR}}"]
required_env: ["PYTHON_BIN", "OUTPUT_DIR"]
min_version: "3.10"
platforms: ["darwin", "linux"]
```

```yaml
# tools.local.md —— 不提交，自动生成
PYTHON_BIN: "/opt/homebrew/bin/python3"
OUTPUT_DIR: "$HOME/.openclaw/output"
```

这样 `tools.md` 是接口文档，`tools.local.md` 是运行时环境描述。

### 2. 用探测脚本生成本地配置

在仓库里放一个 `scripts/tools_env.sh` 或 `tools_check.py`，负责检测 `python3`、`node`、`ffmpeg`、`jq` 等工具的实际路径、版本、平台，然后生成 `tools.local.md`。Agent 每次启动或执行关键任务前调用它刷新，而不是依赖人工手写。

探测脚本输出最好是一份 JSON 报告，包含 `ok`、`warn`、`fail` 三类状态，Agent 可以根据结果决定继续执行还是阻止任务。

### 3. 路径统一使用变量展开

`tools.md` 里不要写死 `/Users/alice/...`，也不要直接写 `~`，因为 `~` 在部分启动环境下不会被展开。统一约定：

- 项目内路径用 `{{PROJECT_ROOT}}` 前缀。
- 用户级路径用 `{{HOME}}` 前缀。
- 由启动器或探测脚本事先展开为绝对路径，再注入 Agent 上下文。

### 4. MCP server 启动参数标准化

每个 MCP server 在 `tools.md` 中声明 `command`、`args`、`env` 模板，实际路径由 `tools.local.md` 提供：

```yaml
mcp_server: filesystem
command: "{{NODE_BIN}}"
args: ["{{MCP_DIR}}/filesystem-server.js", "--root", "{{DATA_DIR}}"]
required_version: ">=18"
```

不要直接写 `npx -y package@version`，因为 `npx` 可能调用系统 Node 而非项目所需版本。把 `NODE_BIN` 和 `MCP_DIR` 交给探测脚本生成。

### 5. 版本与权限校验

在 `tools.md` 顶部增加 `_check` 区块：

```yaml
_check:
  ffmpeg:
    min_version: "6.0"
    platforms: ["darwin", "linux"]
  node:
    min_version: "18"
    platforms: ["darwin", "linux", "win32"]
```

校验脚本读取这些声明，逐项检查实际版本和平台，输出结构化报告。这样新机器 `clone` 后跑一条命令就能知道环境哪里不满足。

### 6. 提交策略

- `tools.md` 提交到 git，作为团队共享的接口规范。
- `tools.local.md`、`.env` 不提交。
- 提供 `tools.example.md` 和 `.env.example`，并写清生成方式。
- CI 中用干净环境跑一次探测和校验，确保配置文件可复现。

## 踩坑点

- **macOS 的 `sed -i`**：Linux 上 `sed -i 's/a/b/' file`，macOS 需要 `sed -i '' 's/a/b/' file`。`tools.md` 里避免写内联 sed，优先用 Python 脚本处理文本。
- **Windows 路径**：反斜杠、盘符、空格路径会直接破坏命令解析。统一使用 POSIX 风格路径或让 Python `pathlib` 处理，不要手写 `C:\Users\...`。
- **`$HOME` 不可靠**：在 cron、launchd、systemd 下 `$HOME` 可能为空或指向 `/var/root`。探测脚本应使用 `os.path.expanduser("~")` 并显式校验结果。
- **Node 版本不一致**：`node -v` 和 `which node` 可能分别指向 nvm 版本和系统版本。探测脚本要同时检查两者，如果不一致则报 `warn`。
- **脚本权限**：Linux/macOS 上需要执行位，Windows 上没有执行位概念。可在 `tools.local.md` 中记录是否需要 `chmod +x`，或由探测脚本自动修复。
- **`.env` 不会被自动加载**：`tools.md` 是文档，不是 shell 脚本，Agent 不会自动 source `.env`。必须在启动器或探测脚本中显式加载环境变量，再注入运行时。

## 可复用建议

- 把 `tools.md` 当接口文档：只写工具名、参数、返回、错误码；环境值一律走变量。
- 固定一个 `tools_check` 命令，输出 JSON，作为 Agent 的“环境体检”。
- 工具分层：`core`（内置）、`project`（项目级）、`user`（用户级），避免全局污染。
- 每个工具增加 `failure_hints`，例如“如果 `ffmpeg` 不存在，请执行 `brew install ffmpeg` 或 `apt install ffmpeg`”。这样 Agent 失败时能给出可读提示。
- 版本锁定：探测脚本生成 `tools.local.md` 时记录实际版本，CI 验证与 `tools.md` 的 `min_version` 是否匹配。

## 总结

`tools.md` 的核心职责是“告诉 Agent 有什么工具、怎么调用、边界在哪”；环境差异应该被隔离在可生成、可校验的本地配置层。分层、探测、校验、生成这四步，能明显减少“我机器上好好的，换台机器就挂”的情况。不要信任一次性手写配置，让它能随时被重新生成和检查，才是可维护的工程化姿势。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/097d26cec485d45b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/901bffbe3f583972.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/d85cbd79700f05f4.png)

