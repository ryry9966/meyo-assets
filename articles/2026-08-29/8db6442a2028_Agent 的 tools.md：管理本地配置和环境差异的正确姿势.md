---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 35120
source: 综合讨论
publishedAt: 2026-08-29
---

## 背景

在 OpenClaw、Agent 或 MCP 工具链里，`tools.md` 经常被当成“工具目录”：列出一堆命令、路径、环境变量，Agent 读取后决定如何调用。它看起来像文档，但很多项目实际会把它当作工具加载配置的来源。

问题在于，一旦需要跨机器协作，或者从 macOS 切到 Linux/Windows，`tools.md` 里只要出现 `/Users/zhangsan/.nvm/versions/node/v20...`、`C:\Users\zhangsan\...`、`python3.11` 或是某个本机 token，整个配置就会变得不可迁移。它很容易从“接口文档”退化成“个人环境快照”。

## 问题

本地配置和环境差异通常集中在四类：

1. **路径差异**：用户目录、项目目录、全局 node/python 包路径。
2. **命令差异**：`python` 与 `python3`、`npx` 与 `npm`、`uv` 与 `uvx`、全局安装位置。
3. **运行环境差异**：端口占用、Docker host 地址、代理设置。
4. **密钥差异**：GitHub token、内网证书、OAuth cookie。

如果把这些内容写死进 `tools.md`，典型症状是：clone 后校验失败、Agent 调用工具时报 `command not found`、路径带空格导致命令断裂，或者 PR 里出现了个人目录和 token。

## 做法/步骤

### 1. 让 tools.md 只描述工具契约

`tools.md` 里不应该出现“某台机器的具体路径”，而应该只描述工具能力、参数和引用变量。例如：

```markdown
- id: github-issues
  description: Create/list issues in current repo
  run: ${GITHUB_CLI}
  env:
    GH_TOKEN: ${GH_TOKEN}
    WORKSPACE: ${WORKSPACE_ROOT}
```

`${GITHUB_CLI}` 在 `.env` 或 `config.local.toml` 里定义：

```
GITHUB_CLI=gh
WORKSPACE_ROOT=.
```

如果需要指定路径，可以写：

```
GITHUB_CLI=/opt/homebrew/bin/gh
```

本地 `.env` 或 `config.local.toml` 加入 `.gitignore`。这样 `tools.md` 可以提交，环境差异留给本地文件。

### 2. 用 profile 覆盖，而不是改主文件

维护一个默认 `tools.md`，再维护一个可选 `tools.local.md`。加载时合并：默认配置先读，本地覆盖后合并。工具项可以增加、禁用或替换命令。不要整份复制改写，只写差异。

### 3. 命令解析交给脚本

如果必须支持多 OS，写一个 `scripts/resolve-command` 或 shell wrapper，内部判断平台返回命令：

```
GITHUB_CLI=$(./scripts/bin-path gh)
```

这样 `tools.md` 仍然保持抽象，不直接写死 `/usr/local/bin/gh` 或 `C:\Program Files\Git\...`。

### 4. 对 MCP server 使用版本化入口

优先用 `npx -y @modelcontextprotocol/server-github@0.1.0` 或 `python -m some_mcp_server`，而不是写全局绝对路径。Python 侧使用项目 `.venv/bin/python`，由启动器注入，避免系统 Python 和 venv 混用。

### 5. 提前校验，而不是运行时失败

加一个 `scripts/doctor` 或 `openclaw tools validate`：解析 `tools.md`，检查每个 `run`/`command` 是否在 PATH 中存在、环境变量是否齐全、MCP server 能否握手。CI 中也跑一遍，防止合入坏的默认配置。

## 踩坑点

- **路径带空格与引号**：不要手工拼 shell 字符串。使用 JSON/YAML 的数组参数，或让执行器使用 `spawn` 而非 `shell: true`。
- **Windows 反斜杠**：路径统一用正斜杠，或交给 `path` 库处理。避免手工做字符串替换。
- **Agent 进程不加载 shell rc**：macOS LaunchAgent、Windows 服务、Docker 容器里，都不会 source `~/.zshrc` 或 `~/.bashrc`。不要依赖交互式 shell 的 PATH。最好用绝对路径或显式传入 PATH。
- **`python` 与 `python3` 混用**：统一用 `python -m module`，或定义 `PYTHON_BIN`。如果同时存在系统 Python 和 venv，必须指向 venv。
- **token 泄漏**：`tools.md` 一旦写了 token，git 历史很难清理。密钥只放 secret store 或环境变量，`tools.md` 中只出现变量名。
- **本地覆盖文件不可见**：有 `tools.local.md` 时，要提供 `--effective` 输出，展示合并后真正生效的配置，否则排障时不知道问题来自哪一层。

## 可复用建议

- 提供 `tools.example.md` 或模板，新环境复制后填入本地值；CI 检查 `tools.md` 是否可解析、本地文件是否被 gitignore。
- 加 `doctor` 命令，循环检查命令、环境变量、版本、MCP 连通性，输出人类可读报告。
- 用 JSON Schema 或 zod/ajv 校验 `tools.md`，避免字段拼写错误。
- README 里写清每个工具的 prerequisites：需要什么运行时、最低版本、需要哪些种子 token。
- 跨机器共享的 MCP server，优先使用容器或远程托管，减少宿主环境差异；本地开发再用 profile 映射到宿主工具。

## 总结

`tools.md` 不是个人备忘，而是 Agent 和协作者共同依赖的工具接口文档。把会变的东西——路径、命令、密钥、运行时——迁移到环境变量、本地 profile 和包装脚本里，只把“工具契约”留在主文件中。配合 `doctor` 校验和 `--effective` 展示，才能真正做到新机器可 onboarding、多人协作不互相踩脚。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/8673139a845be917.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/6fd46ac3ad68d036.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/7fd0173efec0279c.png)

