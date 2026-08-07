---
title: OpenClaw 实战：用 IDENTITY.md 构建可进化的 AI 代理身份
feedId: 32022
source: 综合讨论
publishedAt: 2026-08-07
---

## 背景：为什么静态 prompt 不够用

在 OpenClaw 里配置一个 AI 代理，常规做法是在 agent 的系统提示中写一段固定的角色描述、行为规范、工具使用策略。这在小规模任务或单次对话中足够，但一旦把代理部署为长期运行的服务（比如客服 bot、自动化流程编排、多步骤研发助手），就会暴露三个问题：

- **记忆断层**：代理不会从过去的执行中提炼经验，每次启动都从零开始。
- **规则僵化**：一些边界情况在预设 prompt 里覆盖不到，手动补充成本高且容易遗漏。
- **多实例一致性差**：如果同一个代理以不同配置运行在不同渠道，行为漂移难以管理。

于是我们想要一种方式：让 AI 代理把自己的“人格”沉淀成一个可读、可写、可版本管理的文件，并且允许它在遵循一定约束的前提下，根据运行反馈持续修正这个文件。OpenClaw 生态中，最轻量的实现就是 **IDENTITY.md**。

## IDENTITY.md 的本质

IDENTITY.md 是一个存放在代理工作目录下的 Markdown 文件，它包含以下几块结构化信息：

- **角色声明**：描述代理是谁、服务对象、核心目标。
- **行为边界**：允许做什么、禁止做什么（强约束）。
- **当前状态**：最近一次自我总结、活跃上下文、待解决任务。
- **进化记录**：记录了代理在哪些场景下对自身身份做过调整，以及调整原因。

最关键的一点：代理通过 OpenClaw 的工具系统（或 MCP server）拥有对该文件的读写权限，因此可以在对话间隙或任务结束后主动更新 IDENTITY.md，实现身份的自我进化。

## 实践步骤

以下以 OpenClaw 0.x 版本 + 本地文件 MCP 为例，给出完整落地方案。

### 1. 创建初始 IDENTITY.md

在项目根目录或指定的 agent 工作区创建 `identity.md`，内容尽量结构化，方便代理解析和修改：

```markdown
# Agent Identity

## Meta
- version: 1
- last_updated: 2025-03-27T10:00:00Z
- updated_by: human

## Core Profile
- role: 研发环境问答助手
- audience: 后端工程师
- goal: 帮助快速定位配置与部署问题，给出可执行步骤

## Constraints
- 不允许提供生产环境未经审核的配置
- 回答必须基于文档，不虚构 API
- 当不确定时，主动建议查阅源代码

## Current Context
- active_project: openclaw-deployment
- recent_topics: ["docker-compose 编排", "MCP 插件热加载"]
- pending_actions: []

## Evolution Log
- (首次创建，无历史)
```

### 2. 让代理能够读写这个文件

推荐方式是通过 [file-system MCP server](https://github.com/anthropics/mcp-server-file-system) 或 OpenClaw 内置的文件读写插件，赋予代理对工作目录的受限访问权。

配置示例（概念性）：

```yaml
agent:
  name: dev-assistant
  tools:
    - type: mcp
      server: file-system
      args:
        allowedDirectories: ["/workspace/agent-data"]
        allowWrite: true
```

接着在系统提示中明确指导何时以及如何更新 IDENTITY.md：

> 每条对话结束时，当你有新的长期有效发现（如用户偏好、常见问题模式、自身能力缺口），请更新 workspace/identity.md 的 Current Context 或 Constraints 部分。必须保留 Meta 元数据，记录修改时间和原因。

### 3. 配置 OpenClaw agent 引用 IDENTITY.md

在 OpenClaw agent 配置里，将系统提示指向该文件，而非直接内嵌长文本。这样可以确保每次启动都读到最新身份：

```yaml
agent:
  system_prompt: "file://workspace/identity.md"
```

或者更稳健的做法：让 agent 每次都通过工具读取文件获取身份，这取决于你的提示设计。对于需要初始加载的场景，直接引用文件路径是最简单的。

### 4. 触发进化

代理会在以下时机主动提议更新 IDENTITY.md：

- 用户连续三次纠正相同类型问题 → 添加约束项。
- 发现自身缺失某个高频问答领域知识 → 在 Current Context 中记录，并提示人类补充。
- 长期任务结束后 → 更新活跃项目和待办事项。

进化记录示例：

```markdown
## Evolution Log
- 2025-03-28T15:12:00Z | agent | 新增约束：禁止直接修改生产配置，需先输出计划供审核。原因：用户明确此类场景风险。
```

## 踩坑记录

### 1. 无限进化循环

如果代理将“更新身份”本身当作任务，可能反复修改 IDENTITY.md 导致版本爆炸。解决方案：**强制冷却时间**，比如每个自然日内最多修改一次，且必须附带有实质内容变化。在系统提示中明确：“没有新信息时不得更新”。

### 2. 写入冲突与损坏

并发场景下，多个代理实例同时写同一个文件会损坏结构。应对方法：用单一实例持有 IDENTITY.md 的写权限，其它实例只读；或者使用基于锁的 MCP 工具。简单场景下，直接通过 cron 脚本定时备份 `identity_YYYYMMDD.md`。

### 3. 权限失控导致身份被篡改

若代理能够随意删除自己的硬约束，进化就失控了。实践中，我们在文件头部声明禁止修改的区块（如 Constraints 的顶层必须保留），在系统提示中反复强调这一点，并配合自动化检测脚本（git diff 检查关键字段）。

### 4. 遗忘初始目标

代理可能会逐渐偏向短期便利，例如为了让对话快速结束而弱化安全约束。这需要在进化记录中强制写入修改动机，并在人工审核 checklist 中定期回顾（见下文建议）。

## 可复用建议

- **结构化字段优先**：用 YAML front matter 或明确的 Markdown 标题分隔，便于脚本解析和差异对比。
- **版本控制必做**：将整个 agent workspace 纳入 Git，每次自动提交修改并推送，方便回滚和审计。
- **人工审核门禁**：定期（例如每周）检查 IDENTITY.md 的变化，确认没有软化的安全性约束。可配合 pre-commit hook 检测到关键字段变更时发出告警。
- **分层进化**：将身份拆分为只读的“章程”和可变的“运行状态”两个文件，代理只修改后者，降低风险。
- **结合 MCP 提供外部反馈**：通过单独的 evaluation agent 分析 IDENTITY.md 变更是否合理，输出建议给人类。

## 总结

IDENTITY.md 不是魔术，它只是一个受控的文件，充当代理长期记忆和身份基座。在 OpenClaw 体系里，通过工具赋予代理有限的自我调整能力，比起在系统提示里写死一大段文字，更容易适应持续变化的业务场景。它的价值体现在：

- **缩小提示与行为之间的偏差**：代理能将用户反馈直接转化为约束。
- **降低维护成本**：不再需要每次调试都去修改 agent 配置文件。
- **为多人协作提供一致性基座**：不同人员启动同一代理，行为一致。

如果你的代理需要跑上几天甚至几周，不妨花半个小时把 IDENTITY.md 机制搭起来，让代理学会“成为更好的自己”——但永远记得给它上锁。

---

