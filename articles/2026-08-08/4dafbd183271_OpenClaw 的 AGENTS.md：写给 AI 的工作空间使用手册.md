---
title: OpenClaw 的 AGENTS.md：写给 AI 的工作空间使用手册
feedId: 32110
source: 综合讨论
publishedAt: 2026-08-08
---

# OpenClaw 的 AGENTS.md：写给 AI 的工作空间使用手册

## 背景：AI Agent 为什么需要一份“空间说明书”

在 OpenClaw 的自动化日常里，AI Agent 不像 IDE 插件那样只做代码补全——它会通过 MCP（Model Context Protocol）连接外部工具、遍历文件树、执行 Shell 命令、调用第三方 API。每次新开一个工作空间（workspace），Agent 面临的是一个完全陌生的目录结构、定制化的依赖栈和不确定的运行边界。

如果没有任何引导，Agent 通常会依赖通用知识进行决策。这导致两个极端：要么过于保守（反复向你确认），要么过于激进（误删缓存、错误修改构建脚本）。最终你花在 prompt 工程和纠错上的时间，可能比手动操作还多。

OpenClaw 在 v0.7 中悄悄支持了一个文件：**AGENTS.md**。它就像在工作空间里放了一份“给 AI 的使用手册”，用来向 Agent 声明规则、上下文和执行边界。

## 核心问题：我们到底在规范什么

这并不是把系统提示词写进 markdown 这么简单。你在 AGENTS.md 里需要回答 Agent 的四个关键问题：

1. **我是谁？** 当前 workspace 的目标角色（是代码审查助手，还是运维自动化 worker）。
2. **我能碰什么？** 哪些目录可以读/写/执行，哪些必须保持只读。
3. **我怎么执行任务？** 是优先使用 MCP 插件，还是优先读取本地脚本；是单步确认还是连续执行。
4. **边界在哪里？** 禁止操作的类型（如禁止全局安装、禁止修改 `.git` 配置等）。

这四类信息一旦缺失，Agent 的表现就会从“智能体”降级为“盲猜的 AI prompt”。

## 做法与步骤：从零开始写一份有效的 AGENTS.md

### 1. 明确角色声明（Role）
```markdown
# AGENTS.md
## Role
You are a DevOps assistant for this NestJS monorepo. Your primary goal is to automate integration test flows and maintain CI health.
```

声明角色不是为了说一句废话，而是为后续所有“行为判断”提供锚点。例如遇到 `package.json` 版本冲突时，角色决定了它是优先建议更新依赖，还是隔离影响范围。

### 2. 建立目录白名单和黑名单
```markdown
## Safe Directories
- `src/**` (read & write)
- `scripts/*.sh` (execute only via npx or approved bash wrapper)
- `.github/workflows/*.yml` (read-only unless explicitly requested)

## Forbidden
- Never touch `docker/` without manual approval.
- Never modify global config under `~/.npmrc` or `~/.config`.
```

Agent 在通过 MCP 执行命令前，会优先检查这份白名单。这是一个低成本的安全网，尤其当你开启了 Auto-Execute 模式时。

### 3. 绑定插件与工具策略
如果你的 workspace 使用了 OpenClaw 的官方或社区插件（如 Slack 通知、JIRA MCP），在 AGENTS.md 中绑定触发条件：
```markdown
## Plugin Rules
- When a CI job fails, automatically fetch last 20 lines of logs from GitHub Actions via MCP.
- If error contains "ECONNREFUSED", propose restarting docker-compose service first.
- Do NOT ping Slack channel before attempting local retry once.
```
这相当于把 prompt 中反复强调的行为固化为文件规则，避免每次对话都要手动交代。

### 4. 定义执行节奏（execution rhythm）
```markdown
## Execution Rhythm
- For modifications in `src/`, batch up to 3 related file changes in one reply.
- For infrastructure changes (docker, CI), always ask for confirmation with diff preview.
- If a command takes >10s, suggest async execution via `nohup` and report PID.
```

节奏感决定了 Agent 是多动还是迟钝。工程化地设定阈值，比笼统说“小心操作”有效得多。

## 踩坑点：AGENTS.md 不是万能的银弹

### 坑1：编码格式歧义
Agent 解析 AGENTS.md 时，如果文件是 UTF-16 或包含 BOM，可能导致规则部分丢失。务必保持 UTF-8 without BOM。可用 `file AGENTS.md` 检查。

### 坑2：隐式冲突
当 AGENTS.md 与系统提示词（如 OpenClaw core prompt）冲突时，优先级并非一成不变。实测中，具体的“Forbidden”规则会覆盖通用角色定义，但模糊的“should”建议容易被忽略。因此，关键禁令用 `Never` 开头，而非 `You should avoid`。

### 坑3：路径解析陷阱
Agent 执行命令时的工作目录是 workspace root，但某些 MCP 插件可能使用自己的上下文。若 AGENTS.md 写了 `scripts/deploy.sh`，必须确保是相对于 workspace root 的路径。建议使用 `$WORKSPACE_ROOT/scripts/deploy.sh` 并配合 OpenClaw 变量。

### 坑4：规则过期
AGENTS.md 是静态文件，而项目依赖经常变化。最常见的翻车场景是：迁移到 pnpm 后，AGENTS.md 仍写着 `npm ci`，Agent 会机械执行错误命令。建议在 CI 中加入 lint 检查，比如检测是否调用已禁用的包管理器。

## 可复用建议

1. **版本控制是你的朋友**。将 AGENTS.md 与项目代码一起纳入 Git。对于不同分支，可以维护不同的 AGENTS.md（例如 `AGENTS.dev.md` vs `AGENTS.prod.md`），在启动时通过符号链接或环境变量指定。
2. **模板化**。对于一个组织内多个类似仓库，抽取出 base AGENTS.md 模板，包含通用的插件规则和安全边界，再通过 overlay 文件添加项目特定内容。OpenClaw 支持通过 `AGENTS_PATH` 环境变量指定额外规则文件，能组合多个 md。
3. **用 checklists 自检**。每当你对工程做重大结构调整（如更改 linter、新增敏感目录），在 PR 里附带一个 checklist：“AGENTS.md 是否需要同步更新？”这个简单的习惯能减少技术债务。
4. **直接在 AGENTS.md 里写示例对话**。这是我个人收益最高的做法。例如加一段：
   ```markdown
   ## Example patterns
   ### Good
   User: "Run integration tests for user service"
   Agent: (reads AGENTS.md) -> switches to monorepo root, runs `npm run test:int -- --project=user-service`, reports results.
   ### Bad
   Agent: "I'm going to run all tests in the repo" (forbidden by directory rule)
   ```
   Agent 对示例的理解往往强于规则列表，这是一种 few-shot 内化。

5. **与 MCP 配置联动**。在 AGENTS.md 中直接引用 MCP server 名，如“Use `mcp__github` to create PR”，能确保 Agent 不绕路，也便于你追溯对话行为。

## 总结

AGENTS.md 不是一份产品说明书，而是你对 AI 工作流的显式编码。它把那些在对话里反复纠正、解释、叮嘱的东西变成持久化的上下文，减轻你作为操作者的认知负荷。在 OpenClaw 这样一个深度集成 MCP 和插件的环境里，善用 AGENTS.md 差不多等于给 Agent 配了一个“岗位 SOP”。

写得越具体，Agent 越可靠；设计得越结构化，迭代成本就越低。如果你正在构建一个会被多个项目复用的 Agent 工作空间，也许现在就可以在根目录下新建一个 AGENTS.md，把它当作你写给下一个自己的技术委托书。

---

