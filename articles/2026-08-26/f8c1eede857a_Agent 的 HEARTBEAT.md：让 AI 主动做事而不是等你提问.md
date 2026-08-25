---
title: Agent 的 HEARTBEAT.md：让 AI 主动做事而不是等你提问
feedId: 34776
source: 综合讨论
publishedAt: 2026-08-26
---

## 背景

很多 OpenClaw / Agent / MCP 工作流有一个共性：能力很强，但角色很被动。用户不问，它不动；用户问一句，它动一下。MCP 工具接了不少，插件也挂了一排，但真正能在后台持续干活的，少之又少。

问题不是模型不够聪明，而是缺少一个稳定的“驱动层”。HEARTBEAT.md 就是干这件事的：它不是心跳协议，也不是什么复杂框架，而是放在 workspace 根目录的一个约定文件，告诉 Agent——当你被定时唤醒、空闲巡检或任务间隙时，应该看哪里、做什么、做到什么程度、哪些绝对不能碰。

## 问题

被动式 Agent 在工程实践里有三个非常具体的故障：

1. **上下文断档**：任务在会话里说了一半，下次打开 Agent 全忘光，原本要检查的队列、要清理的临时文件、要跟进的异常，没人再提。
2. **只响应不巡检**：日志堆积、失败任务过期、磁盘临时文件膨胀、依赖更新提示，这些都需要有人主动看。Agent 不会自己想起来。
3. **主动性失控**：一旦开发者试图让 Agent“主动一点”，它可能开始乱动文件、重复执行命令、产生大量无效输出。没有边界，主动就是灾难。

HEARTBEAT.md 的核心不是让 Agent 更“兴奋”，而是让它的主动行为变成可审计、可暂停、可回滚的工程动作。

## 做法 / 步骤

### 1. 创建 HEARTBEAT.md

把它放在项目或工作区根目录。内容建议包括几个固定字段：

```md
# HEARTBEAT.md

cadence: every 30m
read_only_by_default: true

check:
  - path: ./queue/*.json
    older_than: 15m
    action: summarize

  - path: ./logs/error.log
    pattern: "FATAL|panic"
    action: notify

allowed_actions:
  - git status --porcelain
  - ls -la ./exports
  - du -sh ./tmp

require_confirm:
  - git add .
  - rm -rf ./tmp/*
  - git push

log: ./.heartbeat/activity.jsonl
state: ./.heartbeat/state.json
```

关键点：默认只读。`allowed_actions` 只放低风险、可验证、幂等的命令。有破坏性或不可逆的操作，必须进 `require_confirm`。

### 2. 绑定调度

在 OpenClaw 侧接一个定时触发器，或用一个轻量 watchdog 脚本。每 30 分钟触发一次 heartbeat 命令，让 Agent 读取 HEARTBEAT.md，并按 `check` 清单逐项执行。

示例触发逻辑：

```text
schedule: */30 * * * *
task: read HEARTBEAT.md, execute all read-only checks
```

如果 OpenClaw 支持 MCP 工具，可以把 HEARTBEAT.md 暴露成一个 MCP 资源或工具：Agent 在任务开始前先读 heartbeat，任务结束后写 log。也可以让 heartbeat 工具返回“距离上次心跳已过多久”“有哪些未处理项”。

### 3. 执行约束

Agent 读完后，必须遵守：

- 只做 `allowed_actions` 里列出的动作；
- 遇到需要写操作的，进入 `propose -> confirm -> execute`；
- 每个检查项返回结构化结果，不要只给自然语言总结；
- 把每次心跳的结果写入 log，下一次心跳前先读 log 和 state，避免重复处理。

一个简化的执行输出可以是：

```json
{
  "ts": "2026-04-12T10:30:00Z",
  "check": "queue",
  "matched": 3,
  "action": "summarize",
  "status": "ok"
}
```

这样 Agent 的“主动”就被限制在一个可追踪的范围内。

## 踩坑点

**心跳太频繁导致 token 消耗爆炸。**  
每 5 分钟全量读 workspace，模型上下文会迅速膨胀。建议 30 分钟到 2 小时一次，并用 `state` 文件只读取增量内容。

**危险命令混入 allowed_actions。**  
比如 `git add .`、`rm -rf ./tmp/*` 被当成“低风险”。Agent 可能因为路径拼写错误或幻觉执行出问题。白名单命令必须先 dry-run 或限制具体路径。

**没有日志回环。**  
写了 `activity.jsonl` 但下次心跳不读，Agent 会反复处理同一批队列文件。必须让 log/state 成为下一轮输入，或用文件时间戳、哈希去重。

**心跳变成“假积极”。**  
只写“一切正常”没有任何实质检查。建议每个 check 项强制要求可验证输出，比如 `ls` 结果、文件行数、时间戳，而不是主观判断。

**权限失控。**  
心跳过程中 Agent 调用了未在 HEARTBEAT.md 中声明的工具。系统提示里要明确约束：心跳期间只能使用文件内列出的动作，其他工具一律拒绝。

## 可复用建议

- 一个项目一个 HEARTBEAT.md，不要全局共用。
- 该文件纳入版本控制，修改需要 review。
- 默认只读，最小动作集。
- “主动”分三级：只读自动执行、低风险自动执行并记录、高风险必须确认。
- 检查项尽量结构化，要求输出 JSON 或表格，而不是自然语言段落。
- 在 CI 或定时任务里先跑 `heartbeat dry-run`，确认不会误操作再上自动执行。
- 使用 `.heartbeat/` 目录存 log 和 state，避免污染仓库根目录。

## 总结

HEARTBEAT.md 不是让 Agent 自己决定一切，也不是什么万能自动化方案。它是把“主动巡检”变成一个可审计、可暂停、可回滚的工程约束。对 OpenClaw / MCP 用户来说，最有效的自动化不是工具越多越好，而是让 Agent 在没人提问时也知道：该看哪里、能做什么、不能碰什么。

先把心跳调稳，再谈自主。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/264334912898eebf.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/1a94465a9e6d8454.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/adef515ff04ffaca.png)

