---
title: Agent 的 HEARTBEAT.md：让 AI 主动做事而不是等你提问
feedId: 33010
source: 综合讨论
publishedAt: 2026-08-14
---

## 背景

多数 Agent 默认是“响应式”的：你问一句，它动一下。真正有用的自动化，不只是能回答问题，而是能按自己的节律去发现事情：依赖挂了、磁盘满了、待办过期、配置漂移、队列堆积。但在 OpenClaw 这类环境里，给 Agent 一个常驻的主动任务清单并不容易。每次新会话都会丢失上下文，定时任务往往只触发一次 API 调用，Agent 醒来后不知道“这次该做什么”。

我们在实践中发现，缺的不是模型能力，而是一份稳定的“心跳协议”。

## 问题

具体会遇到几个问题：

- 定时唤醒后，Agent 没有明确检查目标，输出一堆无关内容。
- 每次执行没有状态记录，重复检查同样的事情，浪费 token。
- 自动动作边界不清，要么完全不敢动，要么乱改文件。
- 输出格式不稳定，人很难从日志里判断这次心跳是否健康。

这些问题本质上是同一个：主动任务没有被工程化地固化下来。

## 做法：把主动任务写进 HEARTBEAT.md

HEARTBEAT.md 是放在工作区根目录的一份协议文件，不是普通提示词，也不是文档。它回答四件事：什么时候醒来、醒来后看什么、发现问题怎么做、做完留什么记录。

### 1. 建立最小模板

```markdown
# HEARTBEAT
- interval: every 30m
- mode: read-only
- state_file: heartbeat-state.json

## Check
1. repo status: 是否有未提交改动
2. service health: 核心接口是否 2xx
3. todo age: 是否存在超过 24h 未更新的阻塞项

## On issue
- read-only: 仅记录到 heartbeat-log.md
- confirm-before-write: 修改文件、提交代码、重启服务前必须等待确认

## Output
- HEARTBEAT_STATUS: OK | WARN | ERROR
- LAST_CHECK: ISO 时间
- OPEN_ISSUES: 数量 + 简要列表
```

### 2. 让 Agent 每次醒来先读它

在 OpenClaw 的定时任务或 scheduled session 里，不要只给一句“检查一下项目”。而是在 system prompt 中固定一句：

> 启动后先读取工作区根目录的 HEARTBEAT.md，按其中 interval 和 Check 执行本轮 heartbeat。

这样协议文件可以版本化、独立修改，不依赖调度配置。比把任务全塞进 cron 命令里稳定得多。

### 3. 用状态文件避免重复劳动

让 Agent 把每次结果更新到 `heartbeat-state.json`，至少记录：

- `last_check_at`
- `last_status`
- `open_issues`

下一轮执行前先读状态文件，只处理新增或未恢复的异常，而不是把整个仓库再扫一遍。状态文件保持小，记录摘要而非全文，避免读取时占用过多 token。

### 4. 采用两级动作权限

第一级 `read-only`：读日志、查服务状态、看监控数据。  
第二级 `confirm-before-write`：任何 git commit、文件修改、服务重启，都必须显式获得确认。

这能避免 Agent 在凌晨自动改坏配置而你还在睡觉。先跑一周 shadow mode，只读加记录，确认没有误判后再开放第二级权限。

### 5. 用固定输出格式收尾

每次 heartbeat 最后输出三行固定格式，方便日志收集、通知和下游解析。不要允许 Agent 自由发挥长篇报告，否则很快会变成噪音。

## 踩坑点

- **检查项过多**：HEARTBEAT.md 很容易膨胀成“什么都检查”。3-5 个关键项足够，检查项越多，Agent 越容易把状态文件写脏。
- **定时任务失败无感知**：比没有定时更危险。给 heartbeat 本身加 watchdog：连续 N 次没有状态文件更新，就通知你。
- **把复杂判断留在 Agent 语境里**：Agent 每次醒来上下文有限，复杂判断应拆成脚本或 MCP 工具，heartbeat 只负责调用结果，不负责推理。
- **默认给写权限**：这是最常见的翻车点。自动修改文件、自动提交代码，一旦误判很难追溯。

## 可复用建议

- 把 HEARTBEAT.md 纳入版本控制，和代码一起 review。
- 用 MCP 工具封装常用巡检动作，例如 `get_service_health`、`check_queue_depth`、`list_stale_todos`，heartbeat 只做编排。
- 状态文件保持小，记录摘要而非全文。
- 每周人工抽查一次 heartbeat 日志，删掉过时检查项。
- 团队里约定：任何人发现“我希望 Agent 主动提醒我”的事情，先写进 HEARTBEAT.md，而不是发私聊。

## 总结

HEARTBEAT.md 不是魔法，它只是把“主动”从一次性灵感变成可执行的协议。对 OpenClaw 这类 Agent 环境来说，最缺的通常不是模型能力，而是稳定的唤醒上下文和动作边界。把这两件事写清楚，AI 才能从“等你提问”变成“到点就干活”。

---

