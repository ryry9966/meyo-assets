---
title: 给 OpenClaw 加一层 proactive：从定时触发到受限自动执行的落地笔记
feedId: 35616
source: 综合讨论
publishedAt: 2026-09-01
---

## 背景

多数 AI 助手目前还是请求-响应模式：用户发一句，模型回一句。但在 OpenClaw、Agent、MCP 这类工程化场景里，有大量任务天然不适合等人开口——例如定时巡检日志、资源状态变更后自动提醒、接口异常时先做只读排查、文件落盘后自动触发解析流程。

这些需求本质上是 proactive：由事件或时间驱动，让 agent 主动发起一次任务执行或通知，而不是用户事后追问。

## 问题

proactive 能力落到工程上，难点并不在“模型会不会主动”，而在于以下几条链路：

1. 触发源从哪里来，怎么接入；
2. 事件如何变成 agent 能理解的任务；
3. 执行权限如何收紧，避免误操作；
4. 重复触发、误报、上下文膨胀怎么控制；
5. 出了问题怎么回溯。

如果只是把定时任务里塞一条 prompt 然后直接调用 agent，通常会出现两类结果：要么模型只输出一段分析文本，不执行任何动作；要么因为权限过宽或事件重复，造成通知风暴甚至错误写入。

## 做法 / 步骤

我目前用的方式是把 proactive 拆成三段：**trigger → task → action/notify**，中间加队列和日志。

### 1. 定义触发源

根据场景选合适的方式：

- 定时：systemd timer 或 cron；
- 文件变化：inotify / 文件系统 watcher；
- HTTP 回调：webhook 接收外部系统事件；
- MCP 资源更新：轮询 MCP resource 或订阅变更。

每个触发源只负责产生标准化事件，不直接调用 agent。

### 2. 事件预处理

事件进入队列前先做规范化，统一字段：

```
event_id, source, type, entity, timestamp, payload
```

同时做两件事：

- 去重：按 `source + type + entity + 时间窗口` 生成指纹，窗口内重复事件只保留一条；
- 冷却：同一指纹在 5 分钟内不重复触发，避免日志异常时形成轰炸。

### 3. 组装 prompt 并调用 agent

不要直接把原始 JSON 全塞进 prompt。只保留关键字段，写成结构化模板，例如：

```text
事件类型：接口错误率超过阈值
实体：payment-service
时间：2025-01-01 10:00:00
关键数据：5xx 比例 12%，持续 3 分钟
可用工具：read_logs, query_metrics

请判断是否需要执行以下动作之一：
- 查询最近 10 分钟错误日志并输出摘要
- 只通知值班人员
- 不处理
```

通过 OpenClaw 的 CLI 或 API 以非交互模式调用 agent，并显式声明 tool allowlist。前期只给只读 MCP 工具，例如 `read_logs`、`query_metrics`，禁止写操作。

### 4. 执行与回落

agent 返回结果后，由外部 runner 决定实际动作：

- 如果返回的是通知意图，走通知通道；
- 如果返回的是只读查询结果，格式化后推送；
- 如果涉及写操作，先进入 dry-run 或请求确认。

每一步都落日志，至少包含：

```
trigger -> normalized_event -> prompt -> model_response -> executed_action -> result
```

## 踩坑点

1. **触发了但不做事**  
   prompt 里没有明确给出可执行动作，模型只输出“建议排查”。解决方法是把动作枚举写清楚，并设置默认动作，例如“无异常则不处理”。

2. **重复触发风暴**  
   没有 event_id 幂等和冷却，一个持续异常会每 20 秒触发一次，通知瞬间淹没。必须在进入 agent 前就做去重。

3. **上下文膨胀**  
   把整个原始 payload 塞进 prompt，模型要么忽略关键字段，要么上下文超限。只提取必要字段，必要时先做摘要。

4. **权限过宽**  
   proactive 任务在没有人工监督时执行，直接给全套工具很容易误操作。每个任务独立配置最小权限集合，写操作默认关闭。

5. **误报疲劳**  
   阈值设置过低，一有抖动就通知，几天后没人再看。应该按严重程度分级，低级别只写日志，不推送。

## 可复用建议

- 先做只读通知类任务，稳定跑一周后再逐步开放写操作。
- 每个 proactive 任务用一个 YAML 描述，包含 trigger、tool allowlist、cooldown、通知级别、超时时间。
- 日志要能回答四个问题：什么时候触发、给了模型什么、模型返回什么、实际执行了什么。
- 关键动作保留 human-in-the-loop：让 agent 提出方案，由人确认后执行。
- 把触发服务和 agent 逻辑分离，触发层只负责事件生产和队列，agent 只负责决策和工具调用，方便独立扩缩容。

## 总结

proactive 不是让模型突然“学会主动”，而是用事件驱动管道和严格权限控制，让 agent 在无人值守场景下可靠地完成巡检、提醒、简单修复。先解决触发、去重、上下文和可观测性，再谈模型判断质量，顺序不要反。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/4c13e107c9b4e25b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/595f8e9799a73176.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/49e986166639abcf.png)

