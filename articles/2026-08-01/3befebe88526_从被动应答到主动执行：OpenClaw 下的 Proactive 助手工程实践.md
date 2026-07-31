---
title: 从被动应答到主动执行：OpenClaw 下的 Proactive 助手工程实践
feedId: 31135
source: 综合讨论
publishedAt: 2026-08-01
---

## 背景：为什么你的 Agent 总在“等人拉闸”

大部分基于 LLM 的助手仍停留在对话式交互：用户发起指令，Agent 执行后返回结果。但在运维、CI 告警、监控巡检等场景里，真正有价值的是“事情发生的那一刻就能被处理”，而不是等用户发现后才开口。我们把这种能力称为 **Proactive（主动式）能力**——Agent 监听事件、理解上下文、自主决策并完成操作，最终把结果推送给需要的人。

对 OpenClaw 社区的 Agent 开发者而言，Proactive 并不是给聊天窗口加一个定时器，而是一套事件驱动、工具编排与安全护栏组合起来的工程机制。

## 问题：从“有人问”到“没人问也要做”的鸿沟

让 Agent 主动工作，表面上看只是加一个 trigger，但在工程落地时会遇到三个核心问题：

1. **上下文如何及时注入**：事件触发时，Agent 需要获取足够的背景信息（日志、最近变更、历史告警）才能做出准确决策，而不是凭一句“出错了”凭空猜测。
2. **执行链路的可控性**：没人盯着的时候，Agent 如果连续调用工具失败或幻觉，造成的噪音甚至破坏会比被动场景更严重。
3. **结果回收与审计**：Proactive 任务跑完不能消失，必须有明确的结果沉淀（工单、日志、IM 通知），并且需要可回溯的决策轨迹。

## 做法：基于 OpenClaw + MCP 的 Proactive Agent 架构

我们选型如下：使用 OpenClaw 作为 Agent 编排框架，通过 **MCP（Model Context Protocol）** 统一管理事件源、知识库和操作工具，再配合一个轻量的调度层实现主动触发。整体架构分为三层：

- **事件源层**：负责采集外部信号，例如 GitHub webhook、PagerDuty 告警、Cron 定时任务。
- **MCP Server 层**：将事件源、诊断工具、通知渠道封装为 MCP 的 Resources 和 Tools。比如 `github-issue-mcp` 暴露最新 issues，`log-query-mcp` 提供错误日志查询。
- **OpenClaw Agent 层**：通过配置 Proactive Policy 将特定事件映射到 Agent 技能；Agent 收到触发后运行“规划-执行-验证”循环，必要时拉起人机确认。

### 步骤

**1. 搭建事件源 MCP 服务**

以 GitHub issue 创建为例，我们用 FastMCP 写一个轻量 server，监听 webhook 后将事件转换为标准化 MCP resource，并通过 URI 如 `github://issues/recent` 供 Agent 读取。关键点是 resource 内容要做**字段裁剪**，只保留标题、标签、body 前 200 字，防止撑爆上下文。

**2. 在 OpenClaw 中定义 Proactive Policy**

在 agent 配置文件里声明 trigger 规则和 plan 模板：

```yaml
proactive:
  - trigger:
      resource: github://issues/recent
      condition: "label = 'bug' and status == 'open'"
    agent: bug_triage_agent
    plan_template: |
      分析该 issue 内容，查询相关错误日志，
      如果日志中有明确报错，尝试给出修复建议并执行创建 PR 的准备工作。
      如果无法确定，则生成诊断摘要并通知值班工程师。
    human_confirm_on_destructive: true
```

这样当新 bug issue 出现时，Agent 便会被唤醒，依据模板展开工作。

**3. 实现 Agent 技能与工具链**

`bug_triage_agent` 可以通过 MCP client 调用 `log-query-mcp` 获取错误栈，再使用 `github-mcp` 的工具搜索相关代码。工具输出全部需要限制 token 数，并增加“摘要”子代理做二次压缩。

**4. 加入安全护栏**

- **重试宽度**：单个任务最多重试 3 次，且退避间隔递增，避免重试风暴。
- **破坏性操作确认**：如需要执行代码提交或重启服务，暂停并请求人在环确认（如发飞书或 Discord 卡片，等待 emoji 反应）。
- **幂等键**：每条 trigger 事件生成唯一幂等键，防止重复消息导致并发执行。

## 踩坑点

**上下文淹没**：最初我们将 GitHub issue body 全量传入，加上日志查询结果，一次对话轻松超过 8k tokens，Agent 开始遗忘最初的触发任务或捏造日志内容。解决方案是强制工具返回摘要而非原始数据，并在 plan 中设置“聚焦当前任务”的提醒。

**幻觉驱动执行**：在一次日志缺失的用例中，Agent 自行脑补了一个“已知的 OOM 错误”，并调用重启服务的工具，被人在环拦截。后来我们在 prompt 中增加了“若信息不足，必须输出人类可读的推断依据并请求补充数据”的规则，并在规划阶段先做证据完整性自检。

**MCP server 断连导致任务卡死**：Proactive 任务长时间运行，一旦某个 MCP 连接断开，Agent 会陷入无限等待。我们在 OpenClaw 外层加了心跳检测和超时清理机制，超时未完成的任务标记为“待复查”并通知。

## 可复用建议

- **把 Proactive 能力当作“自动技能”封装**：每个主动任务定义清晰的 trigger 条件、plan 模板、所需 MCP 资源、回退策略，可以一键部署到不同项目。
- **先做人机协同，再做全自动**：初期让 Agent 每次执行前都请求确认，积累 200 次以上无负面反馈后再降低确认阈值；这样比直接全自动要稳定得多。
- **工具输出做 token 预算**：规定每个工具返回不超过 500 tokens，多余内容由摘要代理压缩，这能极大抑制幻觉。
- **构建事件回放与审计链路**：将所有 trigger 事件、Agent 决策步骤、工具调用结果写入审计日志，方便回溯和离线优化 policy。

## 总结

Proactive 能力不是“AI 帮你自动发几条消息”这么简单，它要求 Agent 在无人干预的情况下完成感官输入、推理、工具使用和结果闭环，同时需要一整套容错、限流与审计机制。借助 OpenClaw 的事件驱动机制和 MCP 的标准化接口，我们可以把这种能力沉淀为可复用的工程资产，让 Agent 真正变得“眼里有活”。

在实际团队中，我们已让 Proactive Agent 分担了约 30% 的初级告警响应和 issue 预处理工作，而工程师只需在需要时登上控制台点一个确认。这正是对“自动化”二字比较诚实的解法。

---

