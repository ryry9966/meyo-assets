---
title: 当 Agent 学会“提前动手”：Proactive 能力的工程化落地
feedId: 31151
source: 综合讨论
publishedAt: 2026-08-01
---

## 背景：被困在“一问一答”里的助手

当前大部分 AI 助手，无论是 Chat 接口还是 Agent 工作流，都默认工作在**被动响应模式**下：用户给出触发指令（消息、API 调用、定时任务），系统执行并返回结果。这种模式在处理查询、对话、手动激活的自动化时足够好用，但在一些场景下明显“慢半拍”：

- 凌晨巡检脚本发现数据库磁盘使用率超过阈值，需要人工看到告警后自己打开助手去执行清理或扩容。
- 代码仓库新的 PR 合并进来却忘了同步更新文档，助手明明能自动识别变更并生成 Draft，却只能在被召唤后动手。
- 用户每天早上的例行“查询日程、天气、推荐通勤路线”其实完全可以提前 5 分钟完成并推送，而不是等用户打开 App 输入“早安”。

**Proactive 能力**，正是让 AI 系统具备“根据上下文或环境变化，在用户开口前就决策并执行有限任务”的能力。它不等指令，而是根据策略主动服务。对于已经具备 OpenClaw、MCP 生态和插件机制的开发者，这不再是空中楼阁——核心挑战变成了工程问题：**如何设计一套既克制又有效、可观测、避免骚扰的主动服务框架。**

## 问题拆解：Proactive 需要哪几块积木

把一个“主动的 Agent”拆开，至少包含三部分：

1. **触发源**：环境变化的信号，比如定时 clock、文件系统 watch、Webhook 事件、数据管道变更 capture。
2. **决策 Agent (Policy Engine)**：对触发信号 + 用户上下文（偏好、历史、当前状态）进行推理，决定“现在该不该动手”、“做什么”、“做到哪一步”。
3. **执行与通知通道**：把结果以非侵入方式交付——可自动执行的直接动手（低风险），需要确认的推 Draft 或摘要。

在 OpenClaw 的体系里，这三部分可以分别映射为 **MCP 服务（提供工具）**、**OpenClaw Agent 的推理与工具编排**、**OpenClaw 的通知插件**。下面给出的方案就是按这个映射落地的。

## 工程实现：基于 OpenClaw + MCP 的 Proactive 流水线

### 1. 构建统一的触发源管理 MCP 服务
先用一个轻量 MCP 服务把各种触发源抽象成标准工具，供 Agent 查询或订阅。例如我们实现一个 `trigger-source-mcp`，暴露：

- `list_active_triggers`：返回当前生效的触发条件（cron 表达式、文件路径 glob、Webhook 源等）。
- `await_trigger_event`：长轮询/阻塞等待一次触发事件。
- `ack_event`：确认事件已处理，避免重复。

触发事件的 payload 统一包含 `source_type`、`timestamp`、`payload`（比如文件变更的 diff 摘要，或 cron 的触发标签）。这个 MCP 服务内部可以集成 `node-cron`、`chokidar`、一个轻量 Express 接收 Webhook，不复杂。

```ts
// 示例工具定义片段
{
  name: "await_trigger_event",
  description: "Wait for a new proactive trigger event and return its payload",
  parameters: {
    timeout_ms: { type: "number", default: 30000 }
  }
}
```

### 2. 设计 Proactive Decision Agent
这是核心。基于 OpenClaw 创建一个 Agent，配置为**后台常驻**（或者由定时唤醒触发），它的系统提示词里明确了“主动服务策略”：

```yaml
# agent-config.yaml 核心片段
agent:
  name: proactive-svc
  model: gpt-4o
  instruction: |
    你是一个主动服务 Agent。你会不断收到触发事件 (trigger_event)。
    对于每个事件，结合用户习惯档案和当前上下文，决定是否采取主动动作。
    如果置信度低于 0.8 或者动作有风险，不直接执行，而是生成一个建议草稿通知用户。
    永远不要连续两次为同一原因骚扰用户。
  tools:
    - trigger-source-mcp  # 获取触发事件
    - user-profile-mcp    # 获取用户偏好，如勿扰时段
    - notification-mcp    # 发送消息、邮件、卡片
    - workspace-mcp       # 允许的操作：写文件、运行脚本等（受限）
  run_mode: loop          # 循环等待事件
```

实际执行时，Agent 的主循环调用 `await_trigger_event` 阻塞，拿到事件后推理，产生“是否行动”的决策。典型的决策树：

- 属于“仅通知”场景（如每日摘要）→ 调用通知 MCP 推送，同时带 `priority: low`。
- 属于“自动修复”场景（如日志目录快满了，可安全归档）→ 直接调用 workspace 工具执行，并通知用户已完成。
- 属于“需确认”场景（如检测到可以优化的大尺寸图片，但删除操作需确认）→ 仅推送建议卡片，等用户回复 “yes” 再动手。

### 3. 通知与反馈闭环
通知不是简单发一条消息，要根据用户状态选择渠道（静默通知、消息卡片、邮件）并支持快速动作。这里可以复用 OpenClaw 已有的通知插件（如 Slack、飞书或者本地消息中心），并为通知增加 `action_id` 参数，允许用户一键批准后续动作，从而形成“建议-确认-执行”的闭环。

## 踩坑记录

1. **触发风暴与骚扰问题**  
   cron每分钟触发，Agent每次都想发通知。必须引入**冷却期**和**指纹去重**：对同类型触发内容计算哈希，一小时内相同哈希不再推送。同时为不同用户配置“勿扰时段”，Agent必须遵守。

2. **决策凭据过少导致“自作聪明”**  
   仅靠触发事件不足以判断上下文。Agent 需要能访问短期记忆（最近对话摘要）和用户档案（显式偏好）。我们搞了个 `user-profile-mcp`，存储诸如“我一般在 9:00-10:00 处理邮件”、“代码目录权限仅限读取”等，Agent 决策前必须先查询。

3. **执行权限边界模糊**  
   Proactive Agent 如果拥有太多执行类工具，一旦误判就很危险。解决方案是**工具分级**：在 MCP 或 OpenClaw 的插件层对工具声明 `risk_level`（low/medium/high），Decision Agent 只被允许调用 low 和部分 medium。high 级操作只能由明确的用户动作触发。

4. **循环等待导致 Agent 资源浪费**  
   持续轮询 `await_trigger_event` 会使 Agent 实例一直占用资源。优化方式：把 trigger-source-mcp 改为支持 WebSocket 推送，或者让 Agent 使用 cron 唤醒（例如每 5 分钟启动一次检查），运行完立刻退出。后者简单可靠，适合中小规模场景。

5. **状态持久化**  
   如果 Agent 崩溃或重启，冷却记录、已处理事件游标不能丢。我们选择在 trigger-source-mcp 中用 SQLite 维护游标和指纹表，Agent 重启后可续接。

## 可复用建议

- **保持 Proactive 策略显式化、可配置**：不要将策略硬编码在 Agent 提示词里。用 JSON/YAML 声明规则：触发条件、允许动作、优先级、冷却期。Agent 加载规则作为知识库的一部分。
- **先做“建议式”，再做“自动式”**：灰度期间只允许 Agent 推送 Draft，积累用户反馈数据后，再逐步放开自动执行。统计回复率，评估策略质量。
- **复用 OpenClaw 插件生态**：触发源 MCP、通知 MCP、用户档案 MCP 都可以抽象为独立插件，方便其他 Agent 复用。社区已有类似的 Webhook 插件和 Cron 插件，改造一下即可。
- **监控与降级开关**：为 Proactive Agent 增加一个简单的 `health` 接口和全局“暂停主动服务”开关。一旦发现骚扰或错误动作，可以在不重启系统的情况下关闭。

## 总结

Proactive 不是要给 AI 装上“偷听的耳朵”，而是让它学会在恰当的时机，以恰当的行动，成为真正“懂上下文”的协作者。通过 OpenClaw + MCP 的混合架构，我们用相对克制的工程投入，就能让 Agent 从“等指令”进化到“看环境+主动建议”，并且在权限、频率、去重上加了足够的保险。下一步，随着 MCP 生态里更多传感器式工具的出现，这种能力会变得更加通用和可依赖。

主动，但绝不冒犯——这才是工程化 proactive 该有的样子。

---

