---
title: Agent 记忆系统设计：让 AI 助手记住偏好，而不是记住噪音
feedId: 33153
source: 综合讨论
publishedAt: 2026-08-14
---

# 背景：为什么加了记忆反而更笨

在 OpenClaw + MCP 的自动化实践里，模型本身是无状态的。每次会话开始，它不知道你习惯用 `uv` 还是 `pip`，不知道你的项目根目录在哪，也不知道上次任务停在哪。于是有人开始把大量历史对话塞进 system prompt，或者把所有聊天切片存进向量库，结果 agent 反而更容易跑偏。

问题不在“记不记”，而在“怎么记”。

# 问题拆解

Agent 记忆常见翻车点：

- 记忆类型混淆：用户偏好、项目事实、过程性对话混在一起。
- 写入无约束：用户随口说一句“今天试试 X”，被当成长期偏好。
- 检索粗糙：只靠向量相似度召回，结果召回的常常是语气词、半截讨论和已失效信息。
- 偏好冲突：同一 key 反复写入，旧偏好覆盖新偏好，无法追溯。
- 只写不删：时间久了记忆变成垃圾场，用户也不知道怎么清理。

# 做法：三层记忆 + SQLite + MCP

## 1. 先分层，别上来就向量库

把记忆分成三类：

- **Profile**：稳定偏好。语言、输出风格、工具偏好、默认路径等。少而精。
- **Facts**：项目/环境事实。仓库路径、服务端口、账号 ID、部署环境等。
- **Episodes**：近期会话摘要。上次做到哪、结论是什么，跨会话复用。

不建议把原始聊天全量当长期记忆。大多数对话只是过程，不是知识。

## 2. 存储从 SQLite 起步

表结构保持简单：

```
memories(
  id,
  type,        -- profile | fact | episode
  scope,       -- global | project | thread
  key,
  value,
  source,      -- user | agent | consolidated
  importance,  -- 1-5
  created_at,
  updated_at,
  metadata
)
```

`scope` 很重要。全局偏好和当前项目事实一定要分开，否则多项目之间会串味。

## 3. 通过 MCP 暴露记忆工具

在 OpenClaw 里做一个最小 MCP memory server，暴露四个工具：

- `remember(type, key, value, scope, importance)`
- `recall(query, scope, limit)`
- `forget(key)`
- `list(scope)`

工具描述要写死：只用于长期偏好和项目事实，不要存临时对话。

## 4. 写入策略：显式优先，隐式谨慎

最稳的方式是让用户显式说“记住我喜欢用 `uv` 而不是 `pip`”，然后 agent 调用 `remember`。

隐式提取可以做，但要加约束：会话结束时由 LLM 输出候选记忆，只保留高 importance、跨会话仍然成立的内容。临时任务、一次性路径、当天试错的配置，不要进 profile。

同一 key 写入时走更新逻辑，更新 `value` 和 `updated_at`，不要无限追加重复记录。

## 5. 检索策略：按需注入，控制 token

进入任务时，按 scope 查询：

- 全局 profile 注入 system prompt，控制在 500 tokens 以内。
- 当前 project 的 facts 按需查询，代码路径、服务端口这类信息优先用精确 key 匹配。
- Episodes 只给最近 1-3 条摘要，不要一上来拉整个历史。

自然语言 query 可以后续加 embedding，但要有阈值截断。SQL 的 `LIKE`、关键词匹配在很多场景已经够用。

## 6. 冲突与确认

遇到同 key 不同 value，新值覆盖旧值，但保留历史版本。如果有冲突且 importance 高，可以先问用户，避免把随口一句话写成硬规则。

# 踩坑点

- **不要把所有对话切片存向量库**：会召回大量过程性内容，污染上下文。
- **不要把用户随口说的“今天试试 X”记为长期偏好**：需要确认或高重要性阈值。
- **记忆注入要可追溯**：开头标记来源和更新时间，例如 `[long-term memory] 2025-01-01: ...`，不要和系统指令混在一起，否则模型可能把记忆当命令执行。
- **MCP server 要注意 scope 隔离**：多项目、多会话之间不要串。
- **敏感信息不要进记忆**：token、密码、私钥不要落库，或脱敏存储。
- **必须提供 forget/list**：只写不删的记忆系统很快会被弃用。

# 可复用建议

- 从 SQLite + 一个最小 MCP memory server 开始，不要一上来上向量数据库。
- 给 OpenClaw 配 `/remember`、`/recall`、`/forget` 三个命令，先跑通显式记忆闭环。
- 注入模板里固定分组：Profile / Facts / Episodes，不要混成一大段。
- 每周或每月做一次记忆整理：合并重复 key、删除过期 fact、提升高频 preference 的 importance。
- 向量检索是可选优化，不是记忆系统的基础。SQLite 能覆盖大量真实场景。

# 总结

让 Agent 真正“记住偏好”，核心不是存更多，而是分层、结构化、可审计、可删除。

Profile 解决“你是谁”，Facts 解决“现在在哪个环境”，Episodes 解决“上次做到哪了”。先在 OpenClaw 里跑通 SQLite MCP 记忆的最小闭环，再根据实际召回质量决定要不要加向量检索。

记忆系统的目标不是让 agent 知道一切，而是让它在需要的时候，只拿到必要且正确的上下文。

---

