---
title: Agent 的 HEARTBEAT.md：让 AI 主动做事而不是等你提问
feedId: 33688
source: 综合讨论
publishedAt: 2026-08-18
---

## 背景

多数 Agent 工作流仍然是 request/response 模式：你发一条消息，它执行一轮工具调用，然后继续等你。放在交互式问答里没问题，但一旦进入自动化、巡检、维护类场景，这个模式就会卡住——没人发消息，Agent 就一直闲着。

常见替代方案是 cron 或外部调度器直接执行脚本。问题是脚本缺少上下文判断，遇到“看起来需要修但要先确认”的情况，要么硬编码处理，要么只能发告警等人处理。OpenClaw 里已经可以用 MCP 连接文件系统、Git、数据库和消息通道，我们缺的不是执行能力，而是一份让 Agent 在无人值守时知道“何时该做什么、做到什么程度”的运行手册。

## 问题

让 Agent 主动做事，难点不是定时唤醒，而是：

- 触发条件不明确，Agent 只能乱猜；
- 没有状态记录，每次醒来都像第一次；
- 自动执行边界模糊，怕它改坏东西；
- 任务一多，优先级和依赖关系没地方沉淀。

如果把这些信息塞进 system prompt，会变得臃肿且难以维护；如果写死在插件代码里，改了逻辑还要重新部署。HEARTBEAT.md 的思路是：把“主动运行规则”放回项目目录，让 Agent 每次心跳时读取、判断、执行、回写状态。

## 做法 / 步骤

### 1. 在项目根目录创建 HEARTBEAT.md

它不承载完整项目背景，只写三件事：什么时候醒、醒来做什么、不能做什么。可以先从最小模板开始：

```markdown
# HEARTBEAT.md

interval: 30m
state_file: .heartbeat/state.json
log_file: .heartbeat/log.jsonl

allowed_actions:
  - git status
  - git diff --stat
  - npm test
  - python scripts/check.py

tasks:
  - id: check-deps
    priority: P1
    trigger:
      type: interval
      every: 1h
    action: 运行 npm outdated --json，只总结高优先级更新

  - id: alert-disk
    priority: P0
    trigger:
      type: threshold
      metric: disk_usage
      gt: 85
    action: 汇总磁盘占用到 #ops 频道
```

### 2. 增加状态文件

`.heartbeat/state.json` 用一个 JSON 记录上次执行时间、每个任务的成功/失败次数、重试计数和锁状态。示例：

```json
{
  "last_run": "2026-01-18T10:00:00Z",
  "tasks": {
    "check-deps": { "last_success": "2026-01-18T09:00:00Z", "failures": 0 }
  }
}
```

Agent 每次运行前读取，运行后写入。状态文件要小而稳定，不要塞大段日志。

### 3. 用定时唤醒或 MCP 事件源驱动

在 OpenClaw 中可以用 cron、systemd timer 或插件触发一次“心跳会话”。会话开始时，Agent 被要求先读 HEARTBEAT.md 和 state.json，然后遍历 tasks，满足 trigger 的任务进入待执行队列。关键是要把“读文件 -> 判断条件 -> 执行 -> 回写状态 -> 输出摘要”这条链固定成工作流，而不是让 Agent 自由发挥。

### 4. 增加锁和审计

每次心跳开始时尝试创建 `.heartbeat/lock`，拿到锁才继续。所有自动执行动作写入 `log.jsonl`，至少包含时间、任务 id、执行摘要、是否修改了外部状态。这样即使误操作也能回溯。

## 踩坑点

**心跳过密**：30 分钟一次比较稳，再密会消耗大量 token，而且日志和状态文件会膨胀。紧急场景不要靠轮询，用事件触发更合适。

**任务描述太模糊**：写“检查项目健康”这种话，Agent 可能做很多多余操作。触发条件必须是可验证的，比如“磁盘使用率大于 85%”“某文件 24 小时内没更新”。

**并发心跳导致重复执行**：多个定时器可能同时触发。锁文件或 flock 是必须的，不要依赖 Agent 自觉判断。

**状态文件被写坏**：Agent 有时会输出非 JSON 内容或字段缺失。建议在回写前做 schema 校验，失败就保留旧状态文件并报警，不要覆盖。

**自动执行动作越界**：默认只允许只读命令和指定目录写入。任何删除、外部发送、生产变更都应该走“生成建议 -> 等待人工确认”的路径，除非任务明确标记为 P0 且已经过白名单校验。

## 可复用建议

- 把 HEARTBEAT.md 当成无人值守运行手册，不写业务细节，只写触发条件、执行边界和确认步骤。
- 优先事件触发，轮询只做兜底。文件变化、阈值告警、MCP 推送都比固定间隔更省资源。
- 首次接入时先 dry-run 一轮，让 Agent 只输出“将要做什么”，人工确认后再打开自动执行。
- 模板化 HEARTBEAT.md 和 state.json，放到团队脚手架里，新项目直接复用。
- HEARTBEAT.md 本身纳入 Git 版本控制，任何自动执行规则变更都要走 review。

## 总结

HEARTBEAT.md 本身没有什么魔法，它只是把 Agent 主动运行所需的触发条件、状态边界和执行白名单显式化了。真正的价值在于：Agent 从“等你提问”变成“按节奏巡检和处置”，而且规则可版本化、可审计、可复用。对于已经在用 OpenClaw + MCP 做自动化的团队，这是一层比较轻的工程约束，比硬编码调度器更适合交给 Agent 长期维护。

---

