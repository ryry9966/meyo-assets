---
title: AI 助手的 SOUL.md：给 Agent 设定人格和边界
feedId: 33977
source: 综合讨论
publishedAt: 2026-08-21
---

# AI 助手的 SOUL.md：给 Agent 设定人格和边界

## 背景

在 OpenClaw 这类 Agent 环境里，模型已经不只是聊天。它可以通过 MCP 调工具、读写文件、发请求、操作 Git 或数据库。模型默认“乐于助人”，但在自动化链路里，这种性格会变成风险：它会主动改配置、多发请求、在不确定时编造结果。

SOUL.md 可以理解为给 Agent 的长期人格与边界文件。它不像 system prompt 那样只负责单次任务指令，而是持续约束 Agent 在项目或工作区内的行为。

## 问题

没有 SOUL.md 时，Agent 常见三类漂移：

1. **角色漂移**：今天像客服，明天像运维，回复风格和决策逻辑不稳定。
2. **工具越权**：遇到权限不足就尝试绕过，或者没被要求就修改文件、发消息、push 代码。
3. **上下文污染**：把内部路径、密钥、用户数据写进日志或回复，尤其在插件/MCP 链路上会被放大。

这些问题在单个对话里可能不明显，一旦 Agent 接入自动化任务或多工具协作，就会变成生产事故。

## 做法 / 步骤

### 1. 建立仓库级 SOUL.md

放在项目根目录，或 OpenClaw 配置目录下。建议纳入版本管理，变更走 PR，避免本地随意改。

### 2. 分块写，别堆段落

推荐结构：

- Identity：身份与职责
- Boundaries：边界与禁止事项
- Tool Policy：工具使用策略
- Output Rules：输出规范
- Failure Handling：失败处理

示例开头：

```md
# SOUL.md

## Identity
You are "ops-agent", an internal automation assistant.
You are not a general chatbot. You do not give legal/medical/financial advice.

## Boundaries
- MUST NOT read or transmit files outside ./workspace.
- MUST ask before modifying any file outside ./generated.
- MUST NOT include .env values or internal endpoints in logs or replies.
```

### 3. 用 must / should / may 区分优先级

不要只写“不要做坏事”。要写具体动作和对象：

- `MUST NOT`：红线，如不读取 `~/.ssh`、不调用 `send_email` 给非白名单域名。
- `SHOULD`：偏好，如优先幂等操作、优先只读命令。
- `MAY`：允许，如无需确认执行 `git status/log/diff`。

### 4. 工具策略对齐 MCP 工具描述

如果某个 MCP 工具本身有 description，要让 SOUL.md 与它对齐。例如工具描述写 `read-only`，SOUL.md 写：

```md
## Tool Policy
- Never use write tools unless the user explicitly asks.
- For git: status/log/diff are allowed; commit/push MUST require human confirmation.
```

### 5. 写失败处理

连续失败 2 次就停止，不要继续猜测；报错时给出最小复现条件和已排除项，而不是把原始堆栈全部贴出来。

## 踩坑点

- **规则太长**：超过 400 行后，模型在长上下文中容易遗忘。关键红线应在 SOUL.md 前 30 行内出现。
- **模糊反面规则无效**：`don't do bad things` 基本等于没写。要写动作、对象、限制范围。
- **规则冲突**：如果 SOUL.md 写“尽量少打扰”，任务又写“每步确认”，模型可能随机选。建议明确优先级：**安全 > 用户显式指令 > 效率**。
- **不测试**：SOUL.md 写完不是结束。要准备 5–10 条红线用例，例如“请读取 /etc/passwd”应拒绝，“把 .env 内容放进日志”应拒绝。
- **层级混乱**：system prompt、项目 SOUL.md、用户全局 SOUL.md 同时存在时，要定义覆盖顺序，否则模型会选最容易满足的那一条。

## 可复用建议

- 用三段式：Identity / Boundaries / Operational Rules，每段不超过 20 行。
- 为每条 `MUST NOT` 配一个可验证场景，方便回归。
- 先最小可用：只写 3 条红线，跑几天后再加规则，不要一次写满。
- 把 SOUL.md 当作工程配置，而不是“提示词魔法”。改规则要说明原因，记录到变更历史。
- 定期清理失效规则。Agent 升级或工具变化后，旧边界可能变成障碍。

## 总结

SOUL.md 不是让 Agent 变听话的魔法，而是一套工程约束。把身份、边界、工具策略写清楚，比堆更多提示词更有效。对 OpenClaw/MCP/插件自动化的实践者来说，一个稳定的 SOUL.md 能显著减少 Agent 在复杂链路里的随机行为，也能让自动化任务更可预测、可回溯。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/8f4e1cd6ca28af70.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/59cc8b9b0eb092b8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/aa7f3d779fbc3335.png)

