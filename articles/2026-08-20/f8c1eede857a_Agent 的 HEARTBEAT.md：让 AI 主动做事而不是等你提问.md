---
title: Agent 的 HEARTBEAT.md：让 AI 主动做事而不是等你提问
feedId: 33936
source: 综合讨论
publishedAt: 2026-08-20
---

## 背景

在 OpenClaw 生态里，大部分 Agent 仍然是响应式的：用户丢一个问题，Agent 调 MCP 工具、查数据、给答案，然后结束。但真实自动化场景里，很多工作恰恰不能等人来问——日志巡检、数据同步状态检查、依赖服务健康探测、PR 合入后的回归确认，这些都需要 Agent 主动触发。

单纯加 cron 定时器只能跑固定脚本，无法复用 Agent 的推理和工具调用能力。而直接让 Agent 在定时回调里“自由发挥”，它又常常陷入空转、泛化总结或重复执行。

后来我在几个 OpenClaw 自动化项目里引入了一个轻量约定：**HEARTBEAT.md**。它本质上是 Agent 的“主动行为契约”——告诉 Agent 每隔多久、按什么节奏、主动做什么、做完怎么汇报。

## 问题

没有 HEARTBEAT.md 之前，我们遇到三类典型问题：

1. **触发空转**：定时器到了，Agent 打开会话，读了一堆上下文，最后只回一句“一切正常”或“我检查了系统，没有异常”，但没有任何实际检查动作。
2. **行为不可预测**：同一个 prompt 每次执行结果差异很大，有时它会去调十几个 MCP 工具，有时只读一个文件就结束。
3. **上下文污染**：长期运行的定时任务不断往同一个 session 里追加消息，几轮之后 Agent 开始“记不清自己该干嘛”。

核心原因是：Agent 缺少一份可执行、可审计、可版本化的“主动任务说明书”。

## 做法

### 1. 在项目根目录创建 HEARTBEAT.md

它不写长段分析，只写触发频率、检查项、动作和输出格式。示例：

```markdown
# HEARTBEAT.md

## 触发
- 频率：每 30 分钟
- 触发方式：cron 调用 OpenClaw task

## 检查
- [ ] `git status` 是否有未提交变更
- [ ] `logs/app.log` 最近 5 分钟内是否出现 ERROR
- [ ] `GET https://api.example.com/health` 是否返回 200

## 动作
- 若有未提交变更：生成 diff 摘要，不自动提交，等待人工确认
- 若有 ERROR：提取最后 5 条错误，写入 `heartbeat.log`，标记级别
- 若健康检查失败：重试 1 次，仍失败则通过通知插件推送告警

## 输出
- 结果追加到 `heartbeat.log`
- 异常时合并通知，不逐条轰炸
```

### 2. 让 cron 或 MCP 事件触发 Agent 读取并执行

在 OpenClaw 的定时任务配置里，不要写具体逻辑，只写一个固定的 task：

```
openclaw run --task "read HEARTBEAT.md, execute the checks and actions, then write result to heartbeat.log"
```

这样 Agent 每次都会先读文件，再规划执行。关键变化是：**具体检查项和动作在文件里，不在 prompt 里**。改 HEARTBEAT.md 不需要改定时器配置。

### 3. 独立 session + 文件沉淀

每次心跳执行使用独立 session，避免上下文膨胀。执行结果只沉淀到 `heartbeat.log`，不保留在会话历史里。这样 Agent 的上下文始终是干净的，每次都能重新理解 HEARTBEAT.md。

## 踩坑点

### 1. 自动执行危险操作
早期我让 Agent 在检测到未提交变更时自动 commit 并 push，结果有一次它把测试文件一起提交了。后来改成 **dry-run 优先**：默认只生成 diff 摘要，除非 HEARTBEAT.md 里显式写 `auto_commit: true` 且人工确认过一次。

### 2. 错误循环
如果 Agent 检测到日志 ERROR，尝试修复后失败，下一轮心跳又触发同一个错误，会无限重试。需要设置**最大重试次数和冷却时间**，比如同一类错误 1 小时内只处理 2 次，之后静默并等待人工介入。

### 3. 通知过频
每 30 分钟一次心跳，如果每次都推送“健康检查失败”，很快会把人淹没。**聚合通知**很重要：相同异常 1 小时内只推送一次，后续只追加到日志。

### 4. 上下文偷偷膨胀
有些人喜欢把心跳执行放在长生命周期的 Agent 会话里，想着“它能记住上次检查结果”。实际结果是上下文越堆越多，Agent 开始产生幻觉。正确做法是每次独立 session，需要记忆就写入文件。

## 可复用建议

- **保持 HEARTBEAT.md 短小**：只写可执行项，不写背景解释。超过 50 行就应该拆分成多个 heartbeat 文件或指向具体脚本。
- **版本化管理**：HEARTBEAT.md 改动走 PR，不要直接在服务器上改。
- **与 MCP 解耦**：文件里只定义“需要检查什么”，具体用哪个 MCP 工具让 Agent 自己选。这样换工具不用改心跳协议。
- **记录 last_run 和 status**：每次执行后更新这两个字段，方便审计和排障。
- **用模板生成**：可以在团队内复用同一套 HEARTBEAT 模板，实现“主动 Agent”的标准化。

## 总结

HEARTBEAT.md 并不是什么复杂机制，它只是把一个关键问题显式化：**Agent 主动做事时，做什么、多久做一次、做完怎么说**。它让定时任务从“盲跑脚本”变成“可推理的主动巡检”，同时也避免了上下文膨胀和不可预测行为。

如果你的 OpenClaw 项目里已经接入了 cron 和 MCP，不妨给 Agent 加一个 HEARTBEAT.md。你会发现，它开始像一个真正在值守的协作者，而不是一个等你开口的问答机器。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-20/00c17e2f74adcf7d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-20/a512e8419eac86d2.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-20/764bff955de757a9.png)

