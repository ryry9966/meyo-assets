---
title: OpenClaw 的 AGENTS.md：一份让 AI 读懂你项目的工作空间手册
feedId: 32343
source: 综合讨论
publishedAt: 2026-08-10
---

## 为什么你需要一份 AGENTS.md

过去一年，Cursor、Aider、Copilot Chat、Claude Code 等 AI 编码工具迅速普及。它们都提供了一个“项目上下文”的配置入口，但彼此不互通。你在 `.cursorrules` 里写了架构说明，Copilot 看不到；你在 Aider 的 `CONVENTIONS.md` 里注明了测试命令，Cursor 照样乱敲。

更麻烦的是，当我们开始在 OpenClaw 这类多 Agent 框架里串联自动化流程，一份集中、跨工具可读的“工作空间使用手册”变得必要。OpenClaw 社区践行的 **AGENTS.md** 正是为了填补这个缺口：把它放在项目根目录，所有能读 Markdown 的 Agent 都能从中获取上下文，无论是编码助手、CI 脚本还是你自建的 MCP 工具。

## 一个典型痛点

最近在一个后端微服务项目中，团队同时用 Cursor 编码、用 OpenClaw Agent 触发定时任务、用自定义 MCP Server 做配置校验。AI 对项目结构的理解一直被割裂：编码时不知道任务脚本的入口，自动化 Agent 不知道代码风格约定，总得人工反复纠正。最后我们在仓库根目录放了一个不到 200 行的 `AGENTS.md`，并让所有工具在启动时去 `fetch` 它——问题消失了大半。

## 如何写出有效的 AGENTS.md

### 1. 文件位置与格式
OpenClaw 的约定是：`<workspace>/AGENTS.md`，即项目根目录下的 Markdown 文件。单仓库多模块时，也可以在每个子模块内放置各自的 `AGENTS.md`，Agent 会优先读取最接近当前执行上下文的那个。

### 2. 内容结构（模板）
一份好用的 AGENTS.md 至少包含这几块：

```markdown
# Project: user-service
## Overview
微服务用户模块，负责登录鉴权、用户 CRUD。

## Directory Layout
- `cmd/`：入口，`main.go` 启动 gRPC 服务
- `internal/service/`：业务逻辑
- `internal/repo/`：数据访问，用 GORM
- `deploy/`：Docker Compose 及 K8s 编排

## Tech Stack
- Go 1.22, gRPC, GORM, PostgreSQL, Redis

## Key Commands
- 单测：`go test ./...`
- 生成 proto：`buf generate`
- 本地启动：`make run`

## AI Instructions
- 生成代码时优先使用 `errors.Wrap`，不要裸返回 err
- 数据库查询必须通过 repo 接口，勿直接使用 db.Raw()
- 修改 proto 后必须先运行 `buf generate`
```

**重点：** 把给 AI 看的约束写在 `AI Instructions` 或直接散落在各节里，指令要具体、可验证。不要写“保持代码整洁”这种虚话。

### 3. 让 Agent 读进去
不同的工具接入方式不同：

- **Cursor / Copilot Chat**：通过 `@AGENTS.md` 引用，或将其设为 always-applied context。
- **Aider**：启动时加上 `--read AGENTS.md` 或在 `.aider.conf.yml` 中 `read: [AGENTS.md]`。
- **OpenClaw Agent**：在 Agent 配置里声明 `context_file: AGENTS.md`，框架会在构建 system prompt 时自动注入。
- **自定义 MCP 工具**：简单地用 `fs.readFileSync` 读取后拼接到 prompt 里即可。

踩坑提醒：部分工具会把整个文件塞进每轮对话的 system message，Token 消耗巨大。此时请务必将 AGENTS.md 的篇幅控制在 **200 行以内**，详细文档用链接指向外部，例如 `See: docs/ARCHITECTURE.md`。

## 实操中的 3 个坑与对策

### 坑 1：信息过时比没有信息更糟糕
重构了目录结构但忘了更新 AGENTS.md，AI 照着旧说明生成代码，制造更大混乱。  
**对策**：把 AGENTS.md 纳入代码评审（Code Review）的 Checklist，每次改动目录/命令/约定时，必须同步更新。可以写一个 pre-commit Hook 检查文件修改时间与关键目录是否匹配，做简单提醒。

### 坑 2：多 Agent 理解偏差
不同的模型对同一指令的遵循度差异很大。你写了“错误必须用 errors.Wrap”，但有些模型只认“use errors.Wrap for error handling”。  
**对策**：措辞统一用英文命令式，并且补一个 `Example` 小节，给正面示例和反面示例。比如：

```
Bad:  if err != nil { return err }
Good: if err != nil { return errors.Wrap(err, "failed to get user") }
```

### 坑 3：编码与换行符
在 Windows 和 Unix 之间切换时，AGENTS.md 可能被自动转换换行符，导致部分 Agent 解析异常。  
**对策**：在项目 `.gitattributes` 中锁定 `* text=auto eol=lf`，或直接统一使用 LF。

## 可复用的工程建议

- **分层管理**：如果项目庞大，不要把所有细节塞进 AGENTS.md。让它作为“总目录”，并用链接指向 `docs/rules/*.md`，让 Agent 按需取用。
- **面向 Agent 的可测试指令**：你可以写一个简单的测试场景：用空对话启动 Agent，问它“这个项目的测试命令是什么？”如果它回答正确，说明 AGENTS.md 起效了；如果不正确，调整你的描述。
- **版本控制**：对于重要迭代，可在 AGENTS.md 顶部标记 `<!-- version: 1.2 -->`，并用 Git Tag 关联，便于回溯时 AI 知道自己读的是哪个版本的约定。
- **复用 OpenClaw 生态**：OpenClaw 已经内置了 AGENTS.md 的解析与缓存机制，你可以用它的 `claw context reload` 重新加载，避免重启整个 Agent 进程。

## 总结

AGENTS.md 并不是什么魔法，它只是把工程团队心照不宣的约定，用 AI 也能读懂的方式固化下来。在 OpenClaw 的视野里，它像一张写给 AI 的工作空间说明书——告诉它边界在哪里、工具怎么用、风格怎么定。越早定义，AI 的产出就越接近你期望的样子。

真正落地之后你会发现，这项投资回报最高的地方，不是某个单一 IDE 的补全，而是当多个 Agent 在同一个仓库上协同工作时，它们终于讲同一种工程语言了。

---

