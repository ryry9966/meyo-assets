---
title: AI Agent 权限边界：用风险标签决定问还是自己干
feedId: 33619
source: 综合讨论
publishedAt: 2026-08-17
---

## 背景

接入 MCP 和插件之后，Agent 的能力从“生成文本”扩展到“执行操作”。它可以读文件、改配置、发请求、调用外部服务。能力变强之后，真正的问题不再是“能不能做”，而是“哪些事该自己做，哪些事必须问人”。

在 OpenClaw 这类可编排工具调用的 Agent 项目里，工具来源可能不止一个：本地插件、远程 MCP server、用户自定义脚本。权限边界如果只靠模型自觉，迟早会出事。

## 问题

常见失败有两种：

- 事事都问：连读个文件都要确认，Agent 变成聊天机器人，自动化失去意义。
- 什么都自己干：为了省事放开所有工具，结果误删资源、发错消息、污染环境。

根本原因不是 prompt 写得不够狠，而是权限判断没有结构化。同一个工具会因参数不同而风险完全不同。比如 `delete_file(path=/tmp/test)` 和 `delete_file(path=/home/user/docs)`，工具名一样，风险级别完全不一样。靠自然语言很难稳定约束这类差异。

## 做法/步骤

### 1. 给工具声明风险标签，而不是只记工具名

建议把每个工具动作拆成四类风险：

- `read`：只读，不改变状态
- `mutation_reversible`：可逆写，比如新增、更新、可回滚
- `irreversible`：不可逆，比如删除、覆盖、清空
- `external_side_effect`：外部副作用，比如发消息、支付、发布

同时记录 `scope`：`single` / `batch` / `global`。风险等级由“风险标签 + scope”共同决定。

### 2. 在工具分发层加一个 pre-action guard

不要让模型自己判断安全。工具调用前，先经过 guard。示例策略：

```yaml
rules:
  - risk: read
    decision: allow
  - risk: mutation_reversible
    scope: single
    decision: allow
  - risk: mutation_reversible
    scope: [batch, global]
    decision: ask
  - risk: irreversible
    decision: ask
  - risk: external_side_effect
    decision: ask_with_confirm
```

这个 guard 可以在 OpenClaw 的工具分发处做，MCP 工具统一经过一层 wrapper。模型只负责提出动作，不负责判断安全。

### 3. 确认类型要区分轻重

`ask` 和 `ask_with_confirm` 不同。高危操作不要只问一次“是否继续”，要展示影响范围并二次确认。允许用户授权 scope，比如“仅本次”“仅当前目录”，避免重复打断。

### 4. 只读操作自动放行，但记录日志

`read` 类工具自动执行，同时记录读取路径和参数。这样后续可以审计 Agent 读过什么，出现异常行为可以回溯。

## 踩坑点

- **只按工具名判断**：必须检查参数路径和范围。删除临时文件和删除主目录不是同一件事。
- **批量绕过**：Agent 可能把一次批量删除拆成多次单条删除，每条都是 `single`，绕开 `ask`。需要在 guard 层做短窗口聚合，或对危险工具按会话计数。
- **prompt 约束不可靠**：只在 system prompt 里写“不要删重要文件”没有用，恶意或错误工具调用仍会执行。
- **MCP server 不可信**：外部 MCP server 可以声明自己是 `read`，实际做写操作。接入前要审查工具 schema，并限制 MCP server 的权限面。
- **确认疲劳**：如果 `ask` 太频繁，用户会机械点允许，guard 就失效。只对真正高危的操作打断。

## 可复用建议

- 权限策略外部化，不要塞进 system prompt。策略文件可以版本管理、测试、按环境切换。
- 对 MCP/插件做能力收敛：不要直接暴露整个 server，按任务只开放必要工具。
- 审计所有 `allow` / `ask` / `deny` 决策和参数，保留操作前快照。
- 写测试用例覆盖：路径穿越、批量删除、外部发送、诱导工具调用。
- 定期复盘拒绝和确认记录，调整风险标签。

## 总结

AI Agent 的权限边界不是“信不信任 Agent”，而是“操作风险是否被结构化表达”。把风险标签和 `scope` 放进工具层，guard 统一决策，模型只负责提出动作，不负责判断安全。能自动的自动，该问的问，该拦的拦，自动化才可持续。

---

