---
title: Agent 的 tools.md：本地配置和环境差异的正确管理姿势
feedId: 34125
source: 综合讨论
publishedAt: 2026-08-22
---

## 背景

在 OpenClaw/Agent 这类执行型环境里，模型不只做问答，还会调用本地 CLI、读写文件、连接 MCP server。很多自动化“在我机器上能跑，换台机器就崩”，根因不是模型能力不够，而是环境信息没有被结构化描述。`tools.md` 可以充当 Agent 的环境契约：告诉它当前机器有什么、没有什么、应该怎么用。

## 问题

默认行为最危险。Agent 通常假设：

- `python` 是 3.x
- `npm install` 总是安全
- `/tmp`、`/usr/local/bin` 可用
- 凭据一定在环境变量里

这些假设在 macOS/Linux/Windows、公司机器、容器、远程开发机之间会失效。结果包括：误用系统 Python、把依赖装错位置、读取陈旧 token、调用未启动的 MCP server。

## 做法

### 1. 分层放置 tools.md

建议两个层级：

- 全局：`~/.openclaw/tools.md`，写通用规则，如“优先使用 `pnpm`，不要用 `sudo npm -g`”。
- 项目：`<repo>/.openclaw/tools.md`，写本项目差异，如“使用 Docker Compose 启动 Postgres，端口 5433”。

加载顺序为先全局后项目，项目覆盖全局。这样既保持通用约束，又能处理具体环境。

### 2. 用“可探测事实”代替主观描述

不要写“Node 版本较新”，写：

```markdown
## 环境探测
运行并参考：
- `uname -s` / `$env:OS`
- `node -v`，本项目要求 >=20
- `pnpm -v`，禁止使用 npm
- `docker info` 是否可用
- `echo $PGHOST` / `echo $env:PGHOST`
```

让 Agent 在动手前先执行探测命令，输出标准化键值，再决定路径和命令。

### 3. 明确硬约束

在 `tools.md` 顶部放“硬约束”，防止常见错误：

```markdown
## 硬约束
- 禁止修改 `~/.bashrc`、`~/.zshrc`
- 禁止在项目外创建文件
- 所有密钥通过 `op read` 或环境变量读取，不得写入日志
- Windows 下使用 PowerShell 语法，不假设 bash 存在
```

这些规则比长段说明更有效，Agent 也更容易遵守。

### 4. 对齐 MCP 与本地工具

MCP server 常需要本地参数：浏览器可执行路径、数据库 socket、Python 解释器。`tools.md` 应说明：

```markdown
## MCP 工具
- `filesystem` MCP 根目录：`$PROJECT_ROOT`
- `postgres` MCP 连接：`localhost:5433`，库名 `openclaw_dev`
- `browser` MCP 使用系统 Chrome，路径交给环境变量 `CHROME_PATH`
```

如果 MCP 配置文件和 `tools.md` 冲突，Agent 会犹豫或选错。明确“以 `tools.md` 为准，MCP 参数仅作回退”。

## 踩坑点

1. **把 secrets 写进 tools.md**。任何 token、API key、私钥路径都不要出现，只写读取方式。
2. **硬编码绝对路径**。`/Users/alice/code/foo` 换机器就失效。用 `$PROJECT_ROOT`、`~/.openclaw`、`%USERPROFILE%`。
3. **文件过长被忽略**。Agent 对长上下文中间部分的注意力会下降。`tools.md` 控制在 200 行内，详细文档放到 README 或 `docs/`，`tools.md` 只留执行必需信息。
4. **与 MCP 配置重复冲突**。如果 MCP server 配置里已经指定了命令，`tools.md` 不要重复写相反内容；要么统一，要么声明优先级。
5. **只覆盖单一平台**。写命令时考虑 `sh`、PowerShell、`cmd` 差异；不确定时要求 Agent 先探测 `$env:OS` 或 `uname -s`。
6. **不更新漂移**。环境升级后 `tools.md` 没改，Agent 继续按旧版本执行。建议每次环境变更后跑一次校验。

## 可复用建议

- 制作一个 `tools.template.md`，包含硬约束、探测命令、MCP 对齐表。
- 把 `tools.md` 纳入版本控制，但用 `.env.example` 或外部 secret 管理敏感信息。
- 写一个 `agent-doctor` 脚本，自动探测环境并生成/更新 `tools.md` 中的“本机环境”部分。
- 在 CI 或初始化脚本中校验 `tools.md` 中声明的命令是否存在，提前暴露漂移。
- 对跨平台项目，分别记录 macOS、Linux、Windows 分支，并让 Agent 先判断当前分支。

## 总结

`tools.md` 不是给 Agent 看的说明书，而是给自动化设置的环境契约。管理本地配置和环境差异的关键，不是写得更多，而是写清楚“当前机器的事实”和“不可违反的约束”。把 secrets 隔离、路径相对化、命令可探测、MCP 对齐，Agent 在本地才不至于把环境差异变成隐性故障。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/23e8f7c43f359009.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/47e324f97876f7f1.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/72a38e2508adf2c3.png)

