---
title: HEARTBEAT.md：让 Agent 按心跳主动干活的工程实践
feedId: 33063
source: 综合讨论
publishedAt: 2026-08-14
---

## 背景

多数 Agent 默认是“提问-响应”模式：你问一句，它动一下；会话结束，任务也随之消失。但在 OpenClaw、MCP、插件自动化的场景里，真正有价值的往往是那些不需要人盯着的事：定时整理、巡检、同步、汇总、提醒。仅靠 system prompt 里写“请主动一点”并不可靠，工程上更实际的做法，是给 Agent 一个常驻的“心跳文件”。

## 问题

被动式 Agent 通常有几个明显缺陷：

- 会话结束后，待办和上下文丢失。
- 主动任务没有明确触发点，只能靠人推。
- 多步任务每次都要重新交代背景，效率低。
- Agent 偶尔“自作主张”做了高风险操作，无法审计。

这些问题的本质不是模型不够强，而是缺少一个持久化、可执行、可约束的“工作台”。

## 做法

### 1. 固定文件位置和读取约定

在工作区根目录放一个 `HEARTBEAT.md`。在 OpenClaw 的 system prompt 或 boot 指令里写清楚：

> 每次启动、定时唤醒或收到事件时，先读取 `/workspace/HEARTBEAT.md`，按其中的规则执行。

这个约定一旦固定，Agent 就拥有了跨会话的“肌肉记忆”。

### 2. 设计 HEARTBEAT.md 的结构

建议用 YAML frontmatter 定义触发和权限，正文维护任务队列。一个最小可用的例子：

```markdown
---
heartbeat: every 30m
triggers:
  - on_workspace_open
  - mcp_cron: "*/30 * * * *"
permissions:
  read: ["/workspace/inbox", "/workspace/notes"]
  write: ["/workspace/logs"]
  tools: ["filesystem.read", "filesystem.write", "notify.send"]
---

# HEARTBEAT.md

## Task Queue
- [ ] 整理 inbox 中超过 24h 未处理的文件
- [ ] 检查 notes/drafts 是否有待发布内容
- [ ] 生成昨日工作摘要并发送到通知频道

## Rules
- 每次只处理一个任务。
- 所有写操作必须先 append 到 logs/heartbeat.log。
- 任何删除动作需要人工确认。
```

### 3. 用 MCP/定时任务触发心跳

不要让 Agent 自己空转轮询，浪费 token。更稳的方式是由外部触发：

- 用 MCP cron server 每 30 分钟调用一次 Agent。
- 或用 OpenClaw 平台的定时插件发送一条 `heartbeat` 消息。
- Agent 收到触发后，执行“读文件 → 判断任务 → 执行 → 更新状态 → 汇报”。

### 4. 执行与日志分离

Agent 只处理队列中标记为 `[ ]` 的事项，完成后移动到 `DONE`，进行中的任务写入 `DOING`。日志统一 append 到 `logs/` 目录，不要覆盖 `HEARTBEAT.md` 中的任务队列。

## 踩坑点

- **上下文膨胀**：不要把 HEARTBEAT.md 当聊天记录。塞入过多历史后，Agent 几次唤醒就会开始忽略关键规则。只保留任务队列、约束和少量状态摘要。
- **自然语言队列难以解析**：用 checkbox、YAML 或固定格式维护队列，避免 Agent 对“差不多、有空处理下”这类模糊描述产生漂移。
- **重复执行**：在 `DOING` 状态旁加时间戳，超过阈值才允许其他 Agent 或下一次心跳重新认领。
- **权限过大**：frontmatter 里明确 read/write/tools 白名单，禁止任意 shell，尤其是接文件系统或消息推送时。
- **循环触发**：如果 heartbeat 动作会再次触发心跳，需要设置最小间隔和幂等机制，否则容易陷入无限循环。

## 可复用建议

- **从低风险任务开始**：先做“整理下载目录”“检查草稿”“生成摘要”，不要一上来就接删除或发布。
- **先跑干模式**：加一个 `DRY_RUN=true`，让 Agent 只输出执行计划，不实际写操作。稳定一周后再放开。
- **优先用 MCP 工具**：filesystem、notify、calendar 等工具比裸 shell 更可控，也更容易审计。
- **纳入版本控制**：HEARTBEAT.md 放 git 仓库，每次变更可审查、可回滚。
- **定期 review**：心跳文件不是 set and forget。每周检查队列是否堆积、规则是否仍适用、权限是否需要收缩。

## 总结

HEARTBEAT.md 不是让模型变强的魔法，它本质上是一个持久化的接口，把 Agent 的主动行为从“即兴发挥”变成“有状态、有边界、可回滚”。它能让一个普通 Agent 稳定地做小事，并随着信任累积逐步放权。对于 OpenClaw/Agent/MCP 的实践者来说，这比反复调 prompt 更接近工程化。

---

