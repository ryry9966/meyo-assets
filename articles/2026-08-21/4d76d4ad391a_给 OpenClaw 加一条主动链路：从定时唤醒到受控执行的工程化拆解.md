---
title: 给 OpenClaw 加一条主动链路：从定时唤醒到受控执行的工程化拆解
feedId: 34051
source: 综合讨论
publishedAt: 2026-08-21
---

## 背景

大多数 AI 助手是被动响应：你问一句，它动一下。但运维、监控、信息收集等场景里，很多活儿适合“提前做”——磁盘快满了先收集证据，代码库有异常提交先分析 diff，监控告警先做初步归类。

在 OpenClaw/Agent/MCP 这套环境里，工具调用能力已经比较顺，但 proactive 能力往往需要自己搭。如果没有约束，定时任务很容易变成“定时乱跑”。

## 问题

proactive 不等于自动执行，主要难在三点：

- 触发器太粗：直接定时跑整个 agent，容易重复执行、误判。
- 决策与执行耦合：边想边做，一旦判断错就直接产生副作用。
- 缺少审计与去重：出问题后难以定位，同样的事件反复处理。

## 做法

我踩过几轮坑后，把 proactive 拆成五个部分：**事件源、上下文组装、决策闸门、白名单执行、回执抑制**。

### 1. 事件源

不要只依赖 cron。最简单是 cron 每 30 分钟唤醒，但触发只负责唤醒，不带复杂判断。更好的做法是用文件变化、webhook、消息队列或系统指标阈值。

事件源只产生一个轻量事件：

```json
{
  "event_id": "disk-high-20250621-0930",
  "topic": "disk_usage",
  "source": "node-01",
  "payload": { "usage": 91 }
}
```

### 2. 上下文组装

每次触发时，只把与事件相关的信息塞进 prompt：事件摘要、最近 N 条相关日志、目标资源状态、上次执行结果。设置硬性 token 上限，比如 1500 token。

目的是防止 agent 读太多旧历史后，把几天前的旧计划重新执行一遍。

### 3. 决策闸门

让 agent 输出结构化 JSON，而不是直接调工具：

```json
{
  "should_act": true,
  "reason": "disk usage above threshold",
  "tool_name": "fs.scan_dir",
  "args": { "path": "/data/logs" },
  "risk_level": "low"
}
```

用 JSON schema 做强校验。只有 `should_act=true` 且 `risk_level` 在允许范围内才进入执行。这个阶段不直接调工具，避免“边想边做”。

### 4. 白名单执行

通过 MCP 暴露工具时，我会把工具分成三类：

- `read_only`：只读，比如查目录、读日志。
- `suggest`：生成建议、计划。
- `execute`：真正写入、删除、发布。

proactive 链路默认只能使用 `read_only` 和 `suggest`。例如磁盘使用率超过阈值时，默认只允许 agent 收集目录大小并生成清理建议，而不是直接删除文件。

### 5. 回执与抑制

执行完后把结果写回事件日志，并给触发器打标记。用 `event_id` 做幂等键，同一事件只处理一次；对同一目标对象设置冷静期，比如 30 分钟内不重复处理。

## 踩坑点

- **把 proactive 当 cron**：定时直接跑完整 agent，很容易重复执行或误判。触发、决策、执行最好分离。
- **上下文污染**：全量历史塞进 prompt，agent 会受到旧任务干扰，甚至把几天前的计划重新执行。要按事件裁剪上下文。
- **权限过大**：一旦 proactive 链路拥有写权限，误判会直接改线上状态。先只读，再影子模式，最后才放行低风险写操作。
- **缺少去重**：告警重复推送会导致 agent 重复处理。必须在入口做 `event_id` 幂等。
- **没有反馈**：执行结果不反馈给后续触发，系统会在相同条件下反复判断、反复失败。

## 可复用建议

- **MCP 工具分级**：`read_only / suggest / execute`，proactive 默认只用前两类。
- **事件指纹**：`event_id = hash(topic + source + 时间桶)`，例如 5 分钟桶，保证同一批告警只处理一次。
- **影子模式**：先只记录“如果执行会做什么”，不真操作。跑一周后对比人工处理结果再放开。
- **审批策略**：凡是 `execute` 类工具，proactive 只能生成 plan 或请求 approve，人工确认后才执行。
- **指标监控**：至少统计 `trigger_count`、`decision_yes`、`action_success`、`action_fail`、`suppressed_duplicate`。`action_fail` 持续上升，说明上下文或决策闸门需要收紧。

## 总结

proactive 真正的工程难点不是“定时触发”，而是无人监督时保持克制。把触发、决策、执行、审计拆开后，风险会小很多。

我的经验是：先做只读 + 影子模式，跑一段时间再开放低风险写操作，比一上来就自动修故障可靠得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/eb13d40636cebdd0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/c5f35dca307aaa7a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/ec2330e5848bc596.png)

