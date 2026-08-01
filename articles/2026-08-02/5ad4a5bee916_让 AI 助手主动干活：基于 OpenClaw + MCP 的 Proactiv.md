---
title: 让 AI 助手主动干活：基于 OpenClaw + MCP 的 Proactive Agent 实践
feedId: 31272
source: 综合讨论
publishedAt: 2026-08-02
---

## 背景：为什么 AI 助手大部分时间都在等你开口

目前主流的 Agent 实践，无论是 OpenClaw、LangChain 还是其他框架，绝大多数交互模式都是响应式：用户发一条消息，Agent 执行一次推理，调用工具，然后给出结果。哪怕配上流式输出，本质上也是“你问，我答”。

但在实际生产环境里，这种模式有天然的局限性。运维场景里，我们希望在 CPU 持续飙高超过阈值时，Agent 不是等你打开聊天框问一句“现在服务器健康吗”，而是已经调出日志、定位到可疑进程、给出初步结论，并直接把消息推到你的飞书或是钉钉里。日程管理、数据巡检、合规检查，都存在大量“不需要人主动开口”的任务。

这就是 Proactive Agent 的应用空间——让 AI 助手从“被调用”变成“主动涌现”。

## 问题拆解：Proactive 不止是定时任务

很多人第一反应是：不就加一个 cron 定时触发吗？实际上，工程化的 proactive 能力要复杂得多，需要同时解决四个问题：

1. **可靠的触发源**：不仅是时间驱动，还可能是外部事件（webhook）、阈值条件、状态变更。
2. **上下文持久化**：Agent 默认无状态，主动式任务需要记住上一次的结论、决策与中间数据，否则每次都从零开始推理。
3. **闭环通知**：执行结果必须以可配置的方式送达用户，而不是等用户打开终端查看日志。
4. **抑制噪音机制**：滥发通知的 Agent 比没有 Agent 更糟糕。

接下来，我们基于 OpenClaw 的原生能力和 MCP 工具生态，给出一个可以直接复用的落地路径。

## 做法：构建一个 Proactive 运维巡检 Agent

我们以一个具体任务为例：**每个工作日上午 8 点，自动分析 GitHub 仓库近 24 小时的提交、Issue 与 PR 活跃度，生成一份简报并推送到飞书群**。这个任务完美覆盖了触发、工具调用、上下文处理、通知四个环节。

### 1. 定义定时触发与工作流

在 OpenClaw 中，可以使用 `scheduled` 类型的 trigger 配合 cron 表达式：

```yaml
triggers:
  - id: morning_report
    type: scheduled
    config:
      cron: "0 8 * * 1-5"          # 周一至周五 8:00
      timezone: "Asia/Shanghai"
    target:
      agent_id: repo_analyst
      message: "Run the proactive repository analysis task."
```

这里 `trigger` 会自动向指定 Agent 发送一条触发消息，Agent 的 system prompt 里可以预置任务执行逻辑，无需每次由用户发出指令。

### 2. 接入 MCP 工具获取数据

为 Agent 配置一个 GitHub MCP Server，官方社区已有现成实现。`openclaw.yaml` 中关键配置如下：

```yaml
agents:
  repo_analyst:
    model: deepseek-chat
    system_prompt: |
      You are a proactive repository analyst. When triggered, you should:
      1. Query the repository's commits, issues, and pull requests for the past 24 hours.
      2. Summarize key activities, highlight anomalies.
      3. Output a final report in Chinese, formatted for Feishu message.
    tools:
      - mcp:
          server: github
          command: npx
          args: ["-y", "@modelcontextprotocol/server-github"]
```

触发后 Agent 会自主调用 `list_commits`、`list_issues` 等工具，MCP 层处理鉴权和结构化返回。

### 3. 通知推送

OpenClaw 支持 action 钩子。我们可以在 workflow 里加一个 notification 步骤，或者在 Agent 工具集中提供 `send_feishu_message` 工具（通过自定义 MCP server）。最简单的落地方式是利用 webhook action：

```yaml
on_completion:
  - action: webhook
    config:
      url: "https://open.feishu.cn/open-apis/bot/v2/hook/xxx"
      method: POST
      body_template: |
        {"msg_type":"interactive","card":{"header":{"title":"仓库日检报告"},"elements":[{"tag":"md","content":"$response"}]}}
```

至此，一个主动式 Agent 就跑通了：定时触发 → Agent 推理并调用 MCP 工具 → 完成分析 → 推送报告。

## 踩坑点和解决思路

实际运行中，以下几个坑几乎一定会遇到。

### 时间精度与时区不一致

cron 服务可能默认使用 UTC，而多数业务希望本地时间。务必在 trigger 配置中显式声明 `timezone`，并用 `TZ` 环境变量同步 Agent 容器时区。否则你会在凌晨三点收到“上午 8 点报告”。

### 上下文漂移

Agent 每次被触发都是独立会话，如果不额外处理，它会丢失历史记忆。对于需要对比增量变化的场景（如新增 Issue 数），可以利用 **OpenClaw 的 memory 机制** 或外部状态存储（Redis JSON）。我们将上次巡检结果的关键数据写入 memory，下次触发时先读取：

```yaml
agent:
  memory:
    type: in_memory   # 生产环境建议对接外部存储
    key: repo_analyst_state
```

在 system prompt 中提示 Agent 先读取 memory，再对比当前数据。

### MCP 工具调用失败重试

MCP Server 是外挂进程，网络抖动或 GitHub API 限流会导致工具调用偶发失败。给关键工具调用添加重试逻辑是必要的。可以在 Agent 内部通过 prompt 引导：“若某工具调用返回错误，请等待 10 秒后重试一次，若仍失败则在报告中标记`部分数据缺失`”。更可靠的做法是在 MCP wrapper 层统一处理重试和降级。

### 噪音控制

主动通知最怕泛滥。我们的实践是加一层条件过滤：仅当活跃度较上次变化超过阈值（如新增 Issue 数 > 2）才推送完整报告，否则只写入日志。可以利用 memory 中存储的上一个基线值进行判断，并在 workflow 的 `on_completion` 中加入条件判断脚本。

## 可复用的设计模式

把上述经验抽象为一个 **Proactive Pipeline 模板**，几乎可以直接套用到监控告警、日报生成、合规巡检等任务：

1. **Trigger**：定时、webhook、消息中间件订阅（如 Kafka）
2. **Pre-check**：读取 memory/外部状态，判断是否真正需要执行（条件过滤）
3. **Execution**：Agent 使用 MCP 工具链完成具体任务
4. **Post-process**：格式化、摘要，必要时再次条件判断是否通知
5. **Notify**：通过 webhook 或自定义通道送达用户
6. **State update**：写回 memory/外部存储，为下一次运行留上下文

此外，建议每个 proactive Agent 都暴露一个简单的管理接口（例如通过 `/status` 命令），让用户可以查看上次运行时间、运行结果、下次触发时间。这能极大降低用户的“失控感”，是克制设计的一部分。

## 总结

Proactive 能力不是给 Agent 装个定时器那么简单。它考验的是上下文持久化、工具可靠性、通知策略和人机交互的克制。在当前 OpenClaw + MCP 的组合下，这类实践在工程上已经足够轻量和可控。

越主动，越需要克制。当你搭的 Agent 开始悄悄地把你关心的东西准备好，而不是等你一遍遍地喊它，才是真正能融入工作的助手。

---

