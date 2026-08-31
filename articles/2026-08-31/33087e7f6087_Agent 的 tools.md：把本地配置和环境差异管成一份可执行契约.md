---
title: Agent 的 tools.md：把本地配置和环境差异管成一份可执行契约
feedId: 35507
source: 综合讨论
publishedAt: 2026-08-31
---

## 背景

在 OpenClaw/Agent 项目里，模型调用工具前通常只能看到 MCP 工具描述或插件说明。这些描述往往是静态的，不会告诉你当前机器是 Linux 还是 macOS、Python 是 `python` 还是 `python3`、配置目录在哪、密钥是否已经注入。

结果就是：同一个 Agent，本地能跑，换到容器、CI 或同事机器就失败。失败原因不是模型不聪明，而是它缺少一份“本地环境契约”。

`tools.md` 就是用来补上这一层的：一份短小的、面向工具调用的环境说明文件。

## 问题

常见差异集中在这四类：

- **环境变量缺失**：`DATABASE_URL`、`OPENAI_API_KEY`、`HOME` 没注入，Agent 不检查就直接调用。
- **路径写死**：`~/.config`、`/tmp`、`%APPDATA%` 在不同系统上不一致。
- **二进制差异**：`python` 和 `python3`、`ffmpeg` 是否存在、Node 版本是 20 还是 22。
- **配置漂移**：`.env.example` 和真实 `.env` 不一致，本地开关与测试环境不同。

这些问题不是靠加大 prompt 能稳定解决的，需要一份可复用、可校验的说明。

## 做法

### 1. 写一份最小 `tools.md`

不要写成 README。只列会影响工具调用的内容：

```markdown
# tools.md

## Assumptions
- OS: linux
- Shell: bash
- CWD: repo root

## Required env
- HOME
- OPENAI_API_KEY
- DATABASE_URL

## Binaries
- python3 >= 3.11
- ffmpeg
- sqlite3

## Paths
- config: ${XDG_CONFIG_HOME:-$HOME/.config}/myagent/config.json
- cache: ${XDG_CONFIG_HOME:-$HOME/.config}/myagent/cache

## Platform notes
- Linux: use systemctl
- macOS: use launchctl

## Checks before use
- test -f .env
- command -v ffmpeg
```

### 2. 在 OpenClaw 会话中注入

把 `tools.md` 作为 Agent 的前置上下文：

```text
Read tools.md before using any tool.
If a required env var or binary is missing, stop and report.
Do not assume paths.
```

这样模型在调用工具前，先有“本机约束”的意识。

### 3. 配一个环境自检脚本

`tools.md` 适合人读，但 Agent 也需要机器可执行校验。可以放一个 `bin/envcheck.sh`：

```bash
#!/usr/bin/env bash
set -euo pipefail
: "${OPENAI_API_KEY:?missing}"
command -v python3 >/dev/null || { echo "python3 missing"; exit 1; }
command -v ffmpeg >/dev/null || { echo "ffmpeg missing"; exit 1; }
```

然后在 `tools.md` 中写明：运行工具前先执行 `./bin/envcheck.sh`。

## 踩坑点

- **不要把 `tools.md` 写成大而全文档**。它只回答“模型调用工具前必须知道什么”。
- **不要写死本机用户名**。比如 `/home/alice/...`，换机器就失效。优先用 `${HOME}`、`${XDG_CONFIG_HOME}`。
- **不要放真实密钥**。`tools.md` 只说需要哪个环境变量，值统一从 `.env` 或 secret manager 注入。
- **不要忽略 shell 差异**。如果目标环境是 Windows，bash 脚本和路径写法要单独说明。
- **不要以为 MCP 工具描述已经够用**。MCP 描述通常不会动态感知本机路径，静态描述和真实环境仍可能脱节。
- **本地改了依赖，记得同步 `tools.md`**。否则文档会变成新的漂移源。

## 可复用建议

- 仓库根放通用 `tools.md`，个人差异放 `tools.local.md`，后者进 `.gitignore`。
- 用生成脚本从 `pyproject.toml`、`package.json` 或 `.env.example` 提取依赖和环境变量，减少手写。
- 在 CI 里增加一步：解析 `tools.md` 中的 `Required env`，与真实环境比对，缺了就失败。
- 给 Agent 的规则要明确：“检查失败不得继续”，避免模型自动跳过缺失项硬跑。

## 总结

本地配置和环境差异，不是靠模型“聪明”来消化的，而是靠一份可执行的契约。最小 `tools.md`、环境自检脚本、变量化路径、本地覆盖文件，这四件事做成一个闭环，比引入复杂框架更稳定。

Agent 的可靠，不是每一步都成功，而是失败时知道停在哪里。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/b235ff923628f28b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/5b0b3eccb91f9306.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/a3edd5ed48bff57c.png)

