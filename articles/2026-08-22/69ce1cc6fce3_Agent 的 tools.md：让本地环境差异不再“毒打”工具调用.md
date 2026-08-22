---
title: Agent 的 tools.md：让本地环境差异不再“毒打”工具调用
feedId: 34229
source: 综合讨论
publishedAt: 2026-08-22
---

# Agent 的 tools.md：管理本地配置和环境差异的正确姿势

## 背景：为什么 Agent 总在环境上翻车

团队里用 OpenClaw 跑自动化、接 MCP 工具时，最常见的失败不是逻辑错，而是环境不一致。有人用 macOS，有人用 Linux；Python 有的是 `python3`，有的是 `python`；数据库在同事 A 机器上是 `localhost:5432`，在同事 B 的 Docker 网络里是 `db:5432`；还有人本地挂了代理，导致 agent 访问内网服务时行为完全不同。

Agent 默认会按“标准环境”猜测路径和端口，一旦猜错就会连续重试、误报工具不可用，甚至调用到系统自带但版本错误的 CLI。更麻烦的是，这些差异通常只存在于每个人的脑子里和 `~/.zshrc` 里，版本管理里根本没有。

## 问题：环境信息需要一份“契约”

直接把一堆环境变量塞进 prompt 不现实，因为会过期、太长、且不可审阅。我们需要的是一份结构化的、可版本管理、可生成、可校验的文件——也就是 `tools.md`。它告诉 agent：这台机器上到底有什么工具、路径是什么、网络服务怎么连、有哪些已知怪癖。它不是文档装饰，而是 agent 执行动作前应该读取的本地环境契约。

## 做法：分层 + 自动生成 + 显式引用

### 1. 分层维护

建议拆成两个文件：

- `tools.md`：团队共享，记录项目要求的工具版本、通用路径约定、服务端口规范。提交到仓库。
- `tools.local.md`：本机差异，记录个人机器上的实际路径、Docker 端口映射、特殊 alias。加入 `.gitignore`，或者通过模板生成。

一个最小可用的 `tools.local.md` 结构：

```markdown
## Machine
- OS: macOS arm64
- Hostname: dev-laptop

## Available CLIs
| tool | path | version | fallback |
|------|------|---------|----------|
| node | /opt/homebrew/bin/node | v20.11.0 | |
| python3 | /usr/bin/python3 | 3.12.2 | python3.11 |
| docker | /usr/local/bin/docker | 24.0.7 | podman |
| psql | /opt/homebrew/bin/psql | 16.1 | |

## Network / Services
- Postgres: localhost:5433 (docker-compose)
- Redis: 127.0.0.1:6379
- API mock: http://localhost:8081

## Env vars (non-secret)
- PROJECT_ROOT=/Users/me/work/openclaw-project
- NODE_ENV=development

## Known quirks
- `docker compose` 需要 sudo
- Python 命令必须用 `python3`，没有 `python`
- `sed -i` 需要 `sed -i ''`
```

### 2. 自动生成机器级内容

不要手写所有信息，用脚本生成机器部分，避免漂移：

```bash
#!/usr/bin/env bash
{
  echo "## Machine"
  echo "- OS: $(uname -s) $(uname -m)"
  echo "- Hostname: $(hostname)"
  echo ""
  echo "## CLIs"
  for c in node python3 docker git psql redis-cli; do
    if command -v "$c" >/dev/null 2>&1; then
      echo "- \`$c\`: $(command -v "$c") ($($c --version 2>&1 | head -1))"
    else
      echo "- \`$c\`: NOT FOUND"
    fi
  done
} > tools.local.md
```

配合 `make tools-md` 或 npm script 执行，每次环境变更后跑一次。

### 3. 在 OpenClaw 中显式引用

如果 OpenClaw 支持 system prompt 或自定义指令，可以加入：

> 开始任务前先读取项目根目录 `tools.md` 和 `tools.local.md`。执行任何 CLI 前用 `command -v <tool>` 验证路径；如果工具不存在，立即报告并停止，不要猜测路径或尝试安装。

更工程化的做法是通过 MCP 暴露只读环境信息。例如注册一个 `env://tools` resource，返回结构化 JSON，agent 可以实时查询，而不是依赖静态文件。插件也可以封装一个 `get_tool_env` 工具，提供“查询本机可用工具”的能力。

### 4. 更新机制

把 `tools.local.md` 的生成脚本挂到 git hook 或定时任务。pre-commit 时可以检查 `tools.local.md` 是否比 `.openclaw/env-snapshot` 旧，旧了就提醒更新。CI 中可以校验 `tools.md` 的格式和必填字段，比如要求的工具是否都标注了版本。

## 踩坑点

- **不要把绝对路径提交到共享仓库**。路径差异本就是要隔离的，使用 `${HOME}` 或相对路径，个人路径只放在 `tools.local.md`。
- **不要写入 secrets**。API key、数据库密码、token 一律放到环境变量或 secret manager，`tools.md` 只写“需要 `PG_PASSWORD`，从 `.env` 读取”。
- **文件太长会浪费上下文**。只保留关键差异，详细文档放到 README 或 wiki。建议控制在 200 行以内。
- **Agent 不遵守**。需要在 prompt 开头强调读取契约，并在 `tools.md` 顶部写清：“工具不存在时立即报告，不要猜测路径”。否则 agent 仍会按训练偏好执行。
- **环境变量作用域**。非交互 shell 不会加载用户 `.zshrc` 或 `.bashrc`，agent 启动命令需要显式 `source` 或由启动器注入。`tools.md` 中要标注哪些变量需要额外处理。
- **跨平台命令差异**。例如 `sed -i` 在 macOS 和 Linux 参数不同，可以在 `Known quirks` 里记录本机适用的写法或 wrapper。

## 可复用建议

- **分层**：`tools.md` 共享 + `tools.local.md` 本机，后者加入 `.gitignore`。
- **生成优于手写**：用脚本采集 `uname`、`command -v`、版本信息，减少人为遗忘。
- **每个关键工具写自检命令**：如 `python3 -c "import psycopg"`，方便 agent 在失败前先确认依赖。
- **纳入 code review**：环境变化要像代码一样评审，尤其是新增服务端口或切换包管理器。
- **结合 MCP**：把环境信息作为 MCP resource/tool 暴露，让 agent 实时查询，避免静态文件过期。

## 总结

`tools.md` 的核心价值，是让 agent 不再靠猜测与本地环境交互。它把散落在个人配置里的差异显式化、结构化、可审阅。维护成本最低的方式是“生成 + 分层 + 校验”，而不是追求一次写出完美的环境文档。对 OpenClaw/MCP/插件用户来说，这份契约能让自动化在团队多台机器上的成功率显著提高，也让排障从“玄学”变成“查文件”。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/9166b1f899d15375.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/4b89ceaadb040eda.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/8f9debd109acee34.png)

