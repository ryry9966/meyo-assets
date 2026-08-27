---
title: Agent 的 HEARTBEAT.md：让 AI 主动做事而不是等你提问
feedId: 34953
source: 综合讨论
publishedAt: 2026-08-28
---

## 背景

在 OpenClaw、Agent、MCP、插件自动化的组合里，我们经常把 Agent 配成“问一句动一下”的工具：用户发消息，Agent 调工具，返回结果。即使接入了 cron、MCP server 或插件，很多自动化也只是定时执行固定脚本，并没有真正让 Agent 根据现场情况决定“现在该做什么”。

这类系统缺少一个轻量的“心跳文件”——一份 Agent 每次醒来都会读的值班表。本文将 HEARTBEAT.md 作为这个约定文件，放在工作区根目录，让 Agent 在无人提问时也能完成有限的主动巡检与小步推进。

## 问题

典型的被动 Agent 有三个明显问题：

1. **定时任务不判断**：cron 只能跑固定命令，条件变化后不会自行调整。
2. **上下文断裂**：Agent 不记得上次做到哪里，重复检查或漏掉新事件。
3. **空转等待**：用户忙的时候，Agent 明明可以提前发现异常、整理信息，却一直等提问。

HEARTBEAT.md 解决的不是“全自动完成所有工作”，而是让 Agent 在无人干预时主动发现、上报、执行低风险动作，把重要决策留给人。

## 做法 / 步骤

### 1. 创建 HEARTBEAT.md

放在工作区根目录，内容保持短小、可判断、可执行。一个最小结构如下：

```markdown
# HEARTBEAT.md

interval: 30m
state_file: heartbeat_state.json

## watch
- uncommitted_changes > 2h
- data/incoming 有新文件
- 证书剩余天数 < 14

## triggers
- if uncommitted_changes > 2h 且今天未通知 -> 发通知
- if data/incoming 有新文件 -> 汇总摘要并上报
- if 证书剩余天数 < 14 -> 生成续期提醒

## allowed_actions
- git status, git diff --stat, ls, du, curl

## forbidden_actions
- git push, rm, delete, 任何支付或删除操作
```

关键不是写得全，而是每个触发条件都能被验证为真或假，避免“关注项目进展”这种模糊描述。

### 2. 接入 OpenClaw / Agent

在 OpenClaw 的 system prompt 或项目规则里加入一句：

```text
会话开始或定时唤醒时，先读取 HEARTBEAT.md 和 heartbeat_state.json，
按 watch/triggers 判断是否有需要主动执行的动作。
若没有，写一行 heartbeat log 后退出。
```

定时触发可以用 cron 插件每 30 分钟发送一条空消息，或者用 MCP server 暴露一个 `heartbeat` 工具，由外部调度器调用。

### 3. 状态外置

Agent 每次执行后写 `heartbeat_state.json`，只记录最近一次运行结果：

```json
{
  "last_run": "2025-01-01T10:30:00Z",
  "last_notified": {
    "uncommitted_changes": "2025-01-01T09:00:00Z"
  },
  "last_event_id": "evt-20250101-0930"
}
```

不要把历史日志全部塞回上下文，否则很快会撑爆 token。

### 4. 执行循环

Agent 的每次心跳遵循固定顺序：

1. 读取 HEARTBEAT.md
2. 读取状态文件
3. 逐项检查 watch 条件
4. 对命中的 trigger 执行 action
5. 写回状态
6. 追加一行结构化日志

## 踩坑点

- **触发条件过于模糊**：写成“关注项目进展”之后，Agent 每次都说“没有明显变化”。要把条件改成可检查的事实，例如“`git status --porcelain` 非空且持续超过 2 小时”。
- **状态文件膨胀**：如果把每次 heartbeat 的完整输出都追加到状态文件，几轮后上下文就爆了。只保留 last_run 和去重所需的 event_id。
- **重复通知**：没有幂等机制时，同一个未提交变更可能每小时提醒一次。用 `last_notified` + 时间窗口抑制重复。
- **过度自动**：给 Agent 开放 `rm`、`push`、支付等不可逆权限后，一次误判就不可控。默认只允许只读命令和通知动作。
- **并发冲突**：cron 和手动会话同时触发时，两个实例可能重复执行。用 lockfile 或单实例锁。
- **时区不一致**：cron 环境 TZ 和用户本地时区不同，“今天”会判断错误。建议所有状态时间统一存 UTC，展示时再转换。

## 可复用建议

1. **先跑最小闭环**：只写一个 watch 和一条 action，稳定跑两天再扩展，不要一次性堆十几个规则。
2. **状态文件独立**：HEARTBEAT.md 管“要做什么”，heartbeat_state.json 管“做到哪了”，两者职责分离。
3. **可观测性优先**：每次心跳至少写一行 log，包含时间、命中项、动作结果。问题排障比功能本身更重要。
4. **权限最小化**：默认只读 + 通知，写入和执行需要二次确认或人工批准。
5. **版本管理**：HEARTBEAT.md 和状态文件一起纳入 git，方便回滚到上周的稳定配置。

## 总结

HEARTBEAT.md 不是让 Agent 变成全自动机器人，而是给它一份“值班表 + 交班记录”。它让 Agent 在没有人提问的时候，也能按节奏检查环境、发现异常、主动上报，同时通过状态外置和权限边界避免失控。

如果你已经在用 OpenClaw、MCP 和插件做自动化，不妨从一个 30 分钟的心跳文件开始：让 Agent 先学会主动发现一件小事，而不是每次等你开口。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/a875abe1f528c3bc.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/a1f492b8a94cc553.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/8fd9b0bf87e0dedf.png)

