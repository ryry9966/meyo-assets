---
title: Agent 记忆系统设计：别把偏好塞进 system prompt，做个三层记忆
feedId: 34197
source: 综合讨论
publishedAt: 2026-08-22
---

# Agent 记忆系统设计：别把偏好塞进 system prompt，做个三层记忆

## 背景

跑 Agent 自动化的人常遇到：今天让它“输出用中文，别带 emoji”，明天新开会话又忘了；长期任务里反复纠正“不要用 sudo npm install -g”，它还是照犯。根因不是模型笨，而是记忆全放在上下文里，一旦截断或新会话，偏好就丢。

更麻烦的是很多人用 system prompt 硬编码偏好。那不是记忆，是静态配置。真正记忆系统要能随交互更新、按场景检索、能过期和冲突处理。

## 问题定义

一个可用的 Agent 记忆系统至少解决四件事：

1. 短期工作记忆：当前任务临时状态，结束可丢。
2. 长期事实/偏好：跨会话保留，需持久化。
3. 检索时机：按需注入，不全量塞进上下文。
4. 更新与过期：记忆会被推翻、冲突、过时。

不分层就会变成一个大 JSON，越来越难维护，上下文成本暴涨。

## 做法：三层记忆 + 一个检索入口

### 1. 工作记忆

存在当前会话或任务上下文，不持久化。维护任务目标、中间结果、下一步计划。限制条数，比如最多 20 条，超过由 Agent 自己总结压缩。

### 2. 短期记忆

存最近 N 次交互的关键事实。SQLite 表：

```sql
CREATE TABLE short_memory (
  id INTEGER PRIMARY KEY,
  session_id TEXT,
  content TEXT,
  created_at TEXT,
  expires_at TEXT
);
```

定期清理超过 7 天或会话结束的条目。短期记忆适合跨会话但时间敏感的信息。

### 3. 长期记忆

真正“记住偏好”的地方，拆三类：

- `user_preference`：用户风格、工具偏好、禁忌。例如“注释用中文”、“不要自动执行 rm -rf”。
- `user_fact`：稳定事实，例如“用户 macOS，默认 shell 是 zsh”。
- `project_memory`：项目约定，例如“测试必须用 pnpm test”。

SQLite 字段：`type`、`key`、`value`、`confidence`、`source`、`last_updated`、`expires_at`、`status`。量小别上向量库，SQLite + 关键词/标签足够，好排查。

### 4. 记忆检索与注入

每次请求全量注入不可取。做一个 `memory_search` 工具，由 Agent 按需调用：

- 按 `type` + 关键词过滤；
- 按 `last_updated` 和 `confidence` 排序；
- 只返回 Top-N（如 5 条）。

在 OpenClaw/Agent 场景里，可把记忆做成 MCP server，暴露 `remember`、`recall`、`forget` 三个工具，各插件通过标准 MCP 共用一套记忆。

### 5. 更新与冲突处理

偏好不是只增不减：

- 同类 key 覆盖旧值，更新 `last_updated`；
- 低置信度不覆盖，保留两条并标记 `conflict`；
- 用户明确说“以后不要...”，置信度设高，优先覆盖。

## 踩坑点

**偏好全塞 system prompt**：偏好变化频繁，放 system prompt 每次都要重写配置，且无法按场景选择。

**记忆只增不删**：三个月前的“项目用 Node 14”早过时，不清理会让 Agent 按旧环境执行。加 `expires_at` 和定期复核。

**过度总结**：自动总结长期记忆会丢失细节和矛盾。保留用户原话，总结只是索引，不是唯一事实来源。

**敏感信息无隔离**：记忆里可能有 key、路径、内部域名。避免全量注入第三方模型，必要时脱敏或加密。

**检索只靠模型自觉**：模型经常不调记忆工具。可在系统提示里要求关键步骤前先 `recall`，或在工具层自动做前缀召回。

## 可复用建议

1. 先用 SQLite，后上向量库。几十到几百条偏好 SQLite 足够，且可审计。
2. 记忆工具做成 MCP，统一 `remember/recall/forget` schema。
3. 写入记录 `source`：用户明确说的 > 模型推断。
4. 临时偏好 TTL 7 天，稳定事实 90 天，项目约定跟随项目周期。
5. 每周跑一次 `memory health`：看冲突数、过期数、占用上下文 Top 条数。

## 总结

让 Agent 真正记住偏好，不是堆更多上下文，而是把记忆当独立系统：工作记忆管当前任务，短期记忆管跨会话热信息，长期记忆管偏好事实；通过 MCP 按需召回，定义更新和过期规则。这样记忆不会变成越来越大的 system prompt，而是可维护、可排查、成本可控的工程组件。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/ce9ad9d6da1b18bf.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/a82da1cd7ea64fe2.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/c3265b3e470a1c9a.png)

