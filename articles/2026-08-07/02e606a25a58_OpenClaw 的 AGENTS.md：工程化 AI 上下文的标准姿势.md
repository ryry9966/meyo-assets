---
title: OpenClaw 的 AGENTS.md：工程化 AI 上下文的标准姿势
feedId: 32012
source: 综合讨论
publishedAt: 2026-08-07
---

# OpenClaw 的 AGENTS.md：写给 AI 的工作空间使用手册

## 背景：为什么需要给 Agent 写说明书

在 OpenClaw 这类 Agent 运行时里，每次会话启动或任务下发时，Agent 对项目是一无所知的。我们当然可以在 prompt 里手打“这个仓库用的是 pnpm”、“不要碰 build.gradle”、“测试跑 `./gradlew test`”，但这些信息会随着会话结束消失，下一次又得重来一遍。更糟的是，团队里每个人对项目的约定理解不同，Agent 的行为就会飘忽不定。

Claude Code 有 CLAUDE.md，Copilot 有 `.github/copilot-instructions.md`，而 OpenClaw 选择了 **AGENTS.md**——一个放在项目根目录下的 Markdown 文件，作为 Agent 的长期记忆与行为约束。它本质上是一份 **写给 AI 的工作空间使用手册**，用结构化、可版本化的方式把“项目要怎么和 AI 协作”固化下来。

## 问题：Agent 最常见的三类失控

在没有 AGENTS.md 的项目里，Agent 经常会犯这些错误：

1. **基础操作失误**：不知道包管理器是 yarn 还是 pnpm，随手 `npm install` 生成新的 lock 文件。
2. **环境误判**：分不清哪个是生产配置，修改了 `docker-compose.override.yml` 导致本地端口冲突。
3. **规则缺失**：团队要求所有 PR 必须包含 changelog 片段，Agent 却从未补充；或者擅自重构测试框架。

这些问题不是 Agent 本身能力不足，而是**上下文没有被工程化地注入**。

## 做法：从零开始建立 AGENTS.md

### 1. 文件位置与加载机制
OpenClaw 会在会话初始化时扫描工作目录，寻找 `AGENTS.md`（或 `.agents.md`），将其内容注入到系统 prompt 的固定区域。如果子目录中存在自己的 AGENTS.md，OpenClaw 会按作用域递归叠加——这一点很适合 monorepo。

### 2. 最小可用模板
不要一上来就写成万字长文。Agent 的上下文窗口是有限的，且 token 是要计费的。一个紧凑可用的模板如下：

```markdown
# AGENTS.md

## 项目概览
- 项目：openclaw-demo
- 语言：TypeScript + Go
- 包管理：pnpm (前端), Go modules (后端)

## 目录速览
/agent        - OpenClaw agent 定义
/mcp          - MCP server 代码
/plugins      - 插件实现
/infra        - 容器化与部署配置

## 强制约束
- 不要修改根目录的 `package.json` 或 `pnpm-lock.yaml` 以外的前端依赖文件。
- 后端二进制由 `make build` 生成，禁止手动调用 go build 并移动产物。
- 测试必须通过 `make test` 运行，禁止单独执行 `go test`。
- 所有涉及 API schema 的变更必须同步更新 `docs/api/` 下的 OpenAPI 描述。

## 常用命令
- 全量构建：`make all`
- 启动开发环境：`make dev`
- 执行 MCP 相关测试：`make test-mcp`

## 注意事项
- 开发环境依赖 `.env.local`，由团队成员各自维护，Agent 不应覆盖它。
- 永远不要删除 `.gitkeep` 文件。
```

保持三段式：**概览 + 约束 + 命令**。内容要在 200 行以内，让 Agent 在 10 秒内“理解”项目骨架。

### 3. 与 MCP 和插件的联动
OpenClaw 支持 MCP 工具和插件扩展。如果 Agent 需要频繁查询实时信息（如构建状态、Kubernetes 集群名），不要把这些动态数据写死在 AGENTS.md 里。正确做法是在文件中声明工具路径：

```markdown
## 动态信息获取
- 当前部署环境名：调用 MCP 工具 `deploy current` 查询。
- 测试覆盖率阈值：读取 `quality_gate.yml` 的 `coverage.threshold` 字段。
```

这样既避免了静态文件过期，又引导 Agent 调用正确的工具链。

## 踩坑点

### 坑 1：把 README 当成 AGENTS.md
README 是给人看的，包含徽章、截图、哲学描述。Agent 不需要这些，它要的是可执行的指令。直接把 README 喂进去只会浪费 token，还稀释了关键约束。

### 坑 2：文件太长导致 Agent 选择性忽略
在某个实际项目里，AGENTS.md 写了 800 多行，结果 Agent 在对话中途开始忽略后面的约束，因为它被“挤”出注意力窗口。后来我们强制限定在 150 行以内，核心规则放前面，补充说明放 wiki 链接，问题消失。

### 坑 3：忘记维护导致信息腐化
Agent 严格遵守了一条“已废弃的测试命令”，连续三次构建失败，因为它读取的是 3 周前的 AGENTS.md。解决办法是把 AGENTS.md 纳入 PR checklist：任何改变构建流程、目录结构或测试入口的提交，必须同步更新该文件。

### 坑 4：monorepo 中的覆盖陷阱
父目录的 AGENTS.md 写了“使用 npm”，子项目实际是 yarn 项目，OpenClaw 按作用域合并后出现了两条包管理器的冲突指令。实践中应该让子目录的 AGENTS.md 显式声明“本目录覆盖父级包管理约束”，避免歧义。

## 可复用建议

- **分块标记**：用 HTML 注释 `<!-- AGENT:KEY --> ... <!-- /AGENT:KEY -->` 为 Agent 标注重点区域，方便未来做自动化提取。
- **CI 门禁**：在 `make lint` 或 `pre-commit` 里加入检查：如果指定文件（如 `go.mod`、`Dockerfile`）被修改而 AGENTS.md 未更新，给出警告。
- **动态模板注入**：对于多环境的项目，可以用 `envsubst` 或 OpenClaw 的 `context_template` 功能，在 Agent 启动前从模板生成 AGENTS.md，填入环境特定信息。这样就不必在文件中写死。
- **版本签名**：在 AGENTS.md 末尾加一行 `<!-- generated: 2025-01-15T10:00:00Z -->`，Agent 可以据此判断信息时效性，甚至在提示中告知用户“这份指令可能过时”。

## 总结

AGENTS.md 绝不是又一个需要运维的 Markdown 负担。把它看作 **Agent 的程序化上下文**，一旦工程化，它就能大幅降低 AI 协作中的幻觉比率和沟通成本。我们现在的实践是：每个 repo 里，AGENTS.md 与 README.md 同等重要，它是 AI 时代项目基础设施的一部分。

如果你已经开始在 OpenClaw 上构建复杂的自动化流程，不妨停下来花半小时写一份紧凑的 AGENTS.md——你会发现，Agent 突然“懂”你的项目了。

---

