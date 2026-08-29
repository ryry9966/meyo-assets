---
title: 让 Agent 从“等指令”到“事件驱动”：proactive 能力的工程化落地
feedId: 35224
source: 综合讨论
publishedAt: 2026-08-29
---

# 让 Agent 从“等指令”到“事件驱动”：proactive 能力的工程化落地

## 背景

大部分 Agent 目前仍是 turn-based：用户发消息，模型执行，然后等下一轮。但自动化场景里，很多任务不需要人先开口。CI 失败、磁盘占用超标、依赖出现新 CVE、数据同步完成后需要生成报告，这些事件天生适合由 Agent 主动处理。

在 OpenClaw/Agent/MCP 的实践里，proactive 不是把模型调得更“热情”，而是把“事件 -> 判断 -> 执行 -> 反馈”做成一条可观测、可回滚的链路。

## 问题

直接把 Agent 接到 webhook 或定时任务，常见结果是：误触发多、刷屏、动作不可回滚、权限过大。proactive 的主要风险在于，Agent 没等用户确认就做了写操作。所以设计时先回答三件事：什么事件值得处理？哪些动作可以自动做？出事怎么撤回？

## 做法/步骤

### 1. 先选低风险、可回滚的动作

从“通知/草稿/只读检查”开始，不要一上来就自动合并、删资源、发外网。合适的第一批场景：

- CI 失败时创建一条带日志摘要的 issue 草稿
- 磁盘使用率超过阈值时生成清理建议，但不执行删除
- 新 CVE 命中依赖时，在内部群发一条告警并附上修复分支
- 每日数据同步完成后生成报告草稿

这些动作的共同点：即使误触，影响也有限，且用户可以一键关闭。

### 2. 把事件源标准化

在 OpenClaw 插件或自定义 command 里，统一将事件转成结构化数据：

```
{
  "event_type": "disk_usage",
  "source": "node-03",
  "payload": {"usage": 87, "mount": "/data"},
  "timestamp": "..."
}
```

事件源可以是 crontab、文件监听、GitHub/GitLab webhook、IM 群消息关键字。不要让模型直接读原始 webhook JSON，原始 payload 噪音太大。

### 3. 规则先过滤，LLM 兜底

关键：不要让 LLM 直接决定是否执行危险动作。用确定性规则做 gate，例如：

- 冷却时间：同一 event_type + source 在 30 分钟内只处理一次
- 工作窗口：只在 9:00-22:00 触发
- 阈值：disk_usage > 85% 才进入下一步
- 静默等级：silent / suggest / auto_execute 三档

通过规则后，LLM 只做摘要生成、是否值得通知的判断。这样行为可预测，成本也低。

### 4. 动作封装成 MCP tool / 脚本

执行层尽量用 MCP tool 或幂等脚本。每个 tool 带 dry_run 参数：

```
run_maintenance(host, mount, dry_run=true)
create_issue_draft(title, body, assignee, dry_run=true)
```

这样 shadow mode 和真实执行可以用同一套代码，避免“测试时不会执行、上线后行为不一致”。

### 5. 执行后写反馈和撤销路径

每次主动动作都要输出：

- 触发事件 ID
- 动作 ID
- 判断依据，哪条规则命中
- 结果链接，如 issue、分支、消息
- 撤销方式

例如：

```
action_id: 20250211-001
trigger: disk_usage node-03 /data 87%
decision: rule[disk>85%] + time_window[9-22]
result: created issue draft #142
undo: close #142
```

## 踩坑点

### 坑1：webhook 高频抖动

CI 或监控系统经常重复发事件，Agent 会被反复触发。解决：按 event_type + source 做 fingerprint，冷却 30 分钟到 2 小时。不要每来一条都调 LLM。

### 坑2：直接给写权限

proactive Agent 手里如果是 admin token，误判代价很大。建议先从只读 token + 草稿权限开始。需要自动执行写操作时，限制到单仓库、单目录或单 namespace。

### 坑3：把 proactive 做成“模型自己决定”

有人会写一个长 prompt：“你是一个主动助理，发现有问题就自己处理”。这非常不可控。应改成：事件先过规则，规则通过后才允许模型介入，且模型只能选择预定义动作。

### 坑4：上下文塞太多事件

把所有事件都喂给模型会造成上下文过载，模型开始漏判。只给事件摘要、最近一次同类动作结果、相关路径。

### 坑5：没有撤销路径，用户不信任

一旦用户发现 Agent 做了事却不知道怎么回滚，就会整体关掉。每个动作都要有 undo 或 close 路径，并在消息里明确写出。

## 可复用建议

- 先跑 1-2 周 shadow mode：只记录动作计划，不真实执行，统计准确率和误报。
- 用 OpenClaw 插件/MCP 把事件源和动作解耦，不要在一个脚本里写完所有逻辑。
- 将 proactive 行为做成配置：事件类型、冷却时间、工作时间、最大执行次数、静默等级。
- 所有动作写 audit log，保留 action_id 和 trigger_event_id，便于复盘。
- 设置“全局开关”和“按事件类型开关”，让用户可以随时降级为纯通知模式。

## 总结

proactive 能力的价值不是“模型更聪明”，而是把高频、低风险、可回滚的事件处理前移。真正落地时，控制点不在模型，而在触发、规则、权限和撤销。先把影子模式和干跑做好，再逐步放开自动执行。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/595e85f5c7669992.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/e2ce38f6727a1470.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/c9478808c270267d.png)

