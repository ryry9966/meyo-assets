---
title: Agent 的 HEARTBEAT.md：让 AI 主动做事而不是等你提问
feedId: 32090
source: 综合讨论
publishedAt: 2026-08-08
---

## 为什么 Agent 总在“等你开口”

大多数基于 LLM 的 Agent（包括 OpenClaw 等框架）都是**会话驱动**的——你问一句，它动一次。这在交互式问答、按需调用工具时很自然，但面对定时巡检、周期报告、条件告警等场景就捉襟见肘。每次都要人工触发，不仅低效，还容易遗漏。

工程上真正需要的是：**Agent 能够按时间规律或状态变化主动执行任务**。这并不是让 Agent 突然“有了意识”，而是给它一个可声明、可维护的调度文件，让它按照既定计划自己动起来。我把它称为 **HEARTBEAT.md**——一份放在项目根目录的心跳配置，描述 Agent 应该在什么时间、什么条件下、做什么事。

## 结构：一份可读可配的心跳声明

HEARTBEAT.md 本质是一个 Markdown 文件，用 YAML/JSON 块承载结构化信息，其他部分留给注释和上下文。最小可用的结构如下：

```markdown
# Agent Heartbeat
version: 1
heartbeat:
  interval: 60s       # 轮询扫描间隔

tasks:
  - id: daily-digest
    schedule: "0 9 * * 1-5"
    timezone: Asia/Shanghai
    prompt: >
      请结合今天日历和最近的对话记录，
      生成一份今日工作要点，并通过 Slack
      #daily 频道发送。
    tools: [memory_search, slack_send]
    retry: 3
    timeout: 30s

  - id: cpu-watch
    schedule: "*/10 * * * *"
    condition:
      tool: mcp_server_status
      path: cpu.percent
      op: "> 80"
    prompt: >
      CPU 持续高于 80%，请检查进程并
      发送告警到运维群。
    tools: [mcp_server_status, wecom_bot]
```

这里的关键设计：

- **schedule**：标准 cron 表达式，精确到分钟级，覆盖绝大多数周期性任务。
- **condition**：心跳执行前先检查前置条件，避免无意义的 LLM 调用。
- **tools**：显式声明任务可能需要调用的工具集，即 Agent 必须能访问这些 MCP 工具或插件。

所有配置对人友好，可以直接在编辑器里修改。团队中非开发角色也能通过变更此文件来调整 Agent 行为，发 PR 即可。

## 让 OpenClaw 动起来：插件化执行

如果 OpenClaw 本身不内置 cron 式触发，我们可以把它做实成一个**本地 Agent 插件**。思路如下：

1. 在 Agent 启动时加载 HEARTBEAT.md，解析出任务列表。
2. 起一个轻量调度器（可用 `node-cron` 或 Python `schedule`），按配置的 `interval` 轮询，检查每个任务的 cron 是否命中、condition 是否满足。
3. 符合条件的任务被组装成一次**内部会话**：将 `prompt` 作为用户消息注入，`tools` 约束可用工具集，然后调用 LLM 生成回答并实际执行工具。
4. 记录每次执行的时间戳、结果、token 消耗到本地日志或数据库中，用于去重和成本审计。

一个极简的 Node 插件骨架如下（示意）：

```ts
// heartbeat-plugin.ts
import cronParser from 'cron-parser';
import { loadHeartbeatConfig } from './config';
import { runAgentTask } from './runner';

export function startHeartbeat(agent: OpenClawAgent) {
  const config = loadHeartbeatConfig('HEARTBEAT.md');
  setInterval(() => {
    for (const task of config.tasks) {
      const interval = cronParser.parseExpression(task.schedule, { tz: task.timezone });
      const now = new Date();
      if (interval.fields.second === 0 && now.getSeconds() < config.heartbeat.interval) { // 简易命中判断
        // 条件检查
        if (task.condition && !evaluateCondition(task.condition)) continue;
        // 去重：距离上次执行必须大于最小间隔
        if (!shouldRun(task)) continue;

        runAgentTask(agent, task).catch(err => {
          console.error(`Heartbeat task ${task.id} failed:`, err);
        });
      }
    }
  }, config.heartbeat.interval * 1000);
}
```

关键是 **`runAgentTask`** 内部要隔离上下文，避免污染主会话。可以将任务作为一次性对话，拿到工具调用结果后发送通知即可。

## 踩坑实录

在多个项目中实践后，这些点是实际出过问题的：

1. **时间精度与时间漂移**  
   用 `setInterval` 做轮询，长时间运行会有毫秒级漂移，可能导致某些分钟的 cron 被跳过。最好用**对齐到整秒**的递归 `setTimeout`，或采用每分钟唤醒一次的机制。

2. **去重不严谨导致重复执行**  
   如果 Agent 重启或调度重叠，同一任务可能被运行多次。必须持久化“上次执行时间”。最简单的做法是本地 SQLite 记录 `task_id + schedule_time` 的唯一约束，并检查距离上次成功执行是否已过任务间隔。

3. **条件检查的依赖工具超时**  
   `condition` 里的工具调用（比如查 CPU）本身可能超时或失败。策略应设为：条件检查失败则跳过本次执行，而不是直接执行任务（防止误告警）。可以加一个 `condition.timeout` 和 `condition.on_fail: skip`。

4. **LLM 成本失控**  
   如果 `interval` 太短或 cron 写错，可能高频触发推理请求。必须加全局限制：同一任务最小间隔 5 分钟，所有任务总请求每小时上限。并利用之前的执行结果做去重缓存，避免相同 prompt 重复请求。

5. **热加载配置**  
   线上修了 HEARTBEAT.md 后，Agent 不应重启。插件应 watch 文件变化（`fs.watch`）或定时重新 parse，但注意解析错误时不能崩掉已有调度任务，要优雅降级并告警。

## 可复用建议

- **模板仓库**：将上述 HEARTBEAT.md 结构做成项目模板，包含常见任务示例（日报、监控、备份提醒），团队可直接复制到自己的 Agent 项目中。
- **集成 MCP**：把心跳执行能力封装为一个 MCP 工具，这样任何支持 MCP 的宿主（不仅仅是 OpenClaw）都能用同一份配置实现主动触发。工具名为 `agent_heartbeat_check`，由外部 cron service 调用。
- **幂等设计**：任务 prompt 中应提示模型“如果已发送过今天的摘要，请跳过”，双重保险。并将通知渠道的发送记录也纳入去重判断。
- **分层调度**：对高精度任务（<1 分钟）不要用 LLM 直接处理，先由传统脚本监控，只在需要分析时再调用 Agent，降低成本和延迟。
- **可观测性**：为心跳任务接入日志监控、成功率统计，最好能生成一个 `/heartbeat-status` 的页面，让团队随时知道 Agent 在忙什么。

## 总结

**HEARTBEAT.md 让 Agent 的能力从“问 - 答”扩展到“感知时间与状态 - 自主行动”**。它的本质是一组声明式的、人类友好的时间驱动指令，不增加额外基础设施，仅靠 Agent 自身的插件体系即可运转。对于使用 OpenClaw 及 MCP 生态的工程师来说，这是一种低成本复用、可进化的实践：文件放在仓库里，所有修改天然受版本控制，逐步丰富任务列表，Agent 就会越来越像一个靠谱的 virtual teammate，而不仅仅是一个应答机器。

当下一个工作日早上 9 点，你刚打开电脑就看到 Agent 自动推送的今日工作重点时，就会发现：主动的 Agent 比被动的更有价值。

---

