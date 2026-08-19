---
title: Agent 的 tools.md：把本地配置和环境差异写成可执行契约
feedId: 33894
source: 综合讨论
publishedAt: 2026-08-20
---

## 背景

维护过多个 OpenClaw / Agent 项目后，很多问题会反复出现：同一套 agent 在 macOS 上能跑，切到 Linux 后命令路径失效；MCP server 需要读 GitHub token，但 agent 不知道去哪取，只能瞎试；插件已经升级，agent 仍按旧参数调用。tools.md 的价值，就是给本地工具链做一张可执行的“事实表”。

## 问题

Agent 对工具的理解通常来自模型记忆或 MCP server 暴露的元数据，但本地差异不会被自动感知：

- `rg` 在 macOS 上是 `/opt/homebrew/bin/rg`，在 Linux 上可能是 `/usr/bin/rg`
- Python 虚拟环境可能叫 `.venv`，也可能叫 `venv`
- `OPENAI_API_KEY` 在 `.env` 里，但 agent 不知道缺失时该停止
- 某些工具需要特定 shell 或 sudo，直接调用会失败

这些差异会导致 agent 反复重试、调用错误命令，甚至在错误路径下产生副作用。

## 做法/步骤

### 1. 建立 tools.md 并让 agent 预读

在项目根目录或 `~/.openclaw/tools.md` 放置文件。如果 OpenClaw 配置支持预读文件，把它加入启动或任务切换时的读取列表；如果不支持，可以在系统提示里明确：“执行前先读项目根目录 tools.md”。

### 2. 控制结构，而不是写成长文

建议 tools.md 只保留这些块：

```markdown
# tools.md
- host: dev-mac
- shell: zsh
- python: .venv/bin/python (要求 3.11+)
- rg: /opt/homebrew/bin/rg

- env:
  - OPENAI_API_KEY: 从 .env 读取；缺失则停止
  - GITHUB_TOKEN: 从 1password 读取，不写明文

- mcp:
  - name: filesystem
    cmd: npx -y @modelcontextprotocol/server-filesystem ~/work
    health: npx -y @modelcontextprotocol/server-filesystem --help

- 差异:
  - if linux: python=~/.venv/bin/python; rg=/usr/bin/rg
```

### 3. 把 tools.md 与实际命令做比对

在 agent 规划阶段，先执行 `command -v rg`、`python --version`、健康检查命令，再与 tools.md 中的声明比对。不一致时，优先以近期探测结果为准，并提示用户更新文件。

### 4. 分层管理

- 全局：`~/.openclaw/tools.md` 管本机通用配置
- 项目：仓库根目录 `tools.md` 管项目差异
- 密钥：`.env` 或密钥管理工具，不在 tools.md 里写明文

## 踩坑点

- **写得太长**：agent 会抓不住重点。tools.md 应该是备忘和契约，不是操作手册
- **明文密钥入库**：很容易顺手把 `.env` 里的值粘进去，提交后泄露
- **只有单机信息**：换机器后 tools.md 与实际环境漂移，agent 按错误路径执行
- **agent 不读**：如果没有在 prompt 或 wrapper 里强制先读，模型可能忽略它
- **忽略健康检查**：tools.md 声明的命令已经失效，但 agent 仍反复调用
- **把动态信息写进去**：端口、PID、临时路径不应是 tools.md 的内容，应交给运行时探测

## 可复用建议

- 用简单列表或表格，保证人和 agent 都能快速解析
- 为每个 MCP / 插件写一行 `health` 命令，便于启动前校验
- 在 CI 或本地 `make doctor` 中校验 tools.md 中的命令是否存在
- 敏感信息只描述来源：`从 .env 读取`、`op read`、`pass show`，不写具体值
- 公共部分提交到仓库，私有差异放 `tools.local.md` 并加入 `.gitignore`

## 总结

tools.md 不是万能配置，而是让 Agent 多一层“本地事实源”。真正有效的是把它当成契约维护：短、结构化、可校验、可分层、不存密钥。它解决的不是“工具怎么用”，而是“在这台机器上，哪些工具可用、以什么方式可用”。

---

