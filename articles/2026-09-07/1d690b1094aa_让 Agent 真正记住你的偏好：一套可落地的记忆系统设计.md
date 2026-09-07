---
title: 让 Agent 真正记住你的偏好：一套可落地的记忆系统设计
feedId: 36450
source: 综合讨论
publishedAt: 2026-09-07
---

# 背景

Agent 每次会话都是无状态的。你教过它「提交信息用中文、测试用 vitest、别自动 push」，下个新会话它照样忘。常见解法有两类：把偏好手写进 system prompt，或者把历史对话全量存下来做检索。前者维护成本高，后者经常召回一堆过期、互相矛盾的上下文。这篇帖记录我在自己的 OpenClaw agent 上做记忆层的一版实践，目标是：**记住的确实是偏好，而不是一次性任务细节**。

# 问题

我最初的做法很粗暴：让模型每轮自己判断「这条信息值不值得存」。两周后记忆文件里塞满了垃圾——某次临时改的目录名、一个已废弃的脚本路径、甚至用户随口一句反问。全量注入 prompt 后，模型反而被带偏。归纳下来是三个问题：

1. 写入无门槛，LLM 自由发挥；
2. 无冲突处理，新旧偏好并存时模型随机站队；
3. 读取不分层，什么都注入等于什么都没注入。

# 做法

最终收敛成三层结构，存在本地 SQLite，通过 MCP 工具读写：

**1. 分型存储**
- `profile`：稳定偏好（代码风格、语言、时区、工具链），单条不超百字，常驻 system prompt，总量控制在 2KB 内；
- `episodic`：项目级上下文（当前分支、部署环境），带 TTL，过期自动清理；
- `procedural`：操作习惯（「改完配置要跑 lint」），只在相关工具被调用时按需检索。

**2. 写入：显式工具 + 会话末抽取**
- 给 agent 一个 `memory.save` 工具，schema 强制包含 `type / key / value / confidence / source`；
- 会话结束跑一个轻量抽取 pass，只允许输出符合 schema 的条目，confidence < 0.6 直接丢弃；
- 同一 key 重复出现自动升 confidence——单次出现不算偏好，重复才算。

**3. 读取：分层注入**
- `profile` 全量注入；`episodic` / `procedural` 用 tag + embedding 检索 top-k；
- 冲突不覆盖：新条目 supersede 旧条目并保留历史，注入时只给最新版本；
- 附带一个 `memory.list` 工具，用户随时可以审计 agent 到底记了什么。

# 踩坑点

- **一次性指令污染偏好库**：某次「先别动 tests 目录」被存成永久偏好，之后 agent 一直绕开 tests。解法是抽取 pass 里强制区分「任务指令」和「长期偏好」，任务类一律不落库。
- **矛盾记忆越积越多**：写了个每周 cron 做合并，同 key 冲突时保留 `last_used` 更近、confidence 更高的那条。
- **注入过多反而降智**：profile 从 6KB 砍到 2KB 后，遵从率明显回升。记忆不是越多越好。
- **别存敏感信息**：API key、内网地址一律不进库，抽取 pass 里加一层正则过滤兜底。

# 可复用建议

- 先做「显式 save 工具 + 2KB 常驻 profile」，这一步就能解决 80% 的重复交代问题，再考虑上 episodic；
- schema 里一定要有 `confidence` 和 `last_used`，后续所有清理、合并逻辑都靠这两个字段；
- 记忆文件让用户能直接看、能 diff——透明度决定信任度，也是排障的第一入口；
- 抽取 pass 用小模型就够，别用主力模型干这活，成本差一个量级。

# 总结

记忆系统的难点不在存储选型，而在**写入门槛、冲突消解、读取分层**这三件工程事。让 agent 显式调用工具写记忆、按类型分层读取、定期合并清理，比指望模型「自己记」可靠得多。整套结构一个下午能搭完，建议先只跑 profile 层两周，观察注入效果，再决定要不要扩展。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/2db9d320b29b4d73.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/67bfacc5d7c5554c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/35aecc82de5b1b70.png)

