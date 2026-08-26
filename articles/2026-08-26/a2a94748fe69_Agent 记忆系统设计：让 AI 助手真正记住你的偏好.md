---
title: Agent 记忆系统设计：让 AI 助手真正记住你的偏好
feedId: 34838
source: 综合讨论
publishedAt: 2026-08-26
---

## 背景

很多 Agent 看起来“有记忆”，实际只是把最近几轮对话塞进上下文窗口。会话一长，偏好就开始漂移；下次启动，之前说过的“回复用中文”“代码别过度注释”“默认时区用 Asia/Shanghai”全部丢失。

记忆不是存聊天记录。它应该是一层可检索、可更新、可控制注入的偏好系统。

## 问题

在 OpenClaw / Agent / MCP 场景里，偏好记忆常见四个坑：

1. 偏好散落在多轮对话中，没有结构化，模型只能靠猜。
2. 全量注入记忆会占用大量 context，噪声比信号多。
3. 用户偏好会变化，旧偏好不失效，导致前后冲突。
4. 多用户、多项目、多插件场景下，容易串记忆。

所以需要一个轻量的、工程化的记忆层，而不是继续往 system prompt 里堆自然语言。

## 做法 / 步骤

### 1. 先分层，不急着上向量库

建议把记忆分成三层：

- **短期工作记忆**：当前任务状态，只在本次会话内使用，不落盘。
- **长期偏好**：相对稳定的用户习惯，例如回复语言、代码风格、默认工具链。
- **情景记忆**：与具体项目或任务相关的事实，例如“项目 A 用 PostgreSQL，部署在 Docker Compose”。

第一版优先做长期偏好，因为收益最明显、实现成本最低。

### 2. 用 MCP 暴露 memory tools

不要直接操作文件。把记忆能力做成 MCP server 或 OpenClaw 插件，暴露几个最小工具：

```json
{
  "save_preference": {
    "key": "reply.language",
    "value": "zh-CN",
    "scope": "user",
    "confidence": 0.95,
    "source": "explicit",
    "ttl_days": 365
  }
}
```

建议工具集：

- `save_preference(key, value, scope, confidence, ttl)`
- `get_preference(key, scope)`
- `search_memory(query, limit)`
- `forget_preference(key)`

### 3. 注入策略：按需检索，不做全量注入

每轮请求前，先根据当前用户输入做轻量关键词匹配，检索 top-k 条相关偏好。通常取 3~5 条就够，不要超过 8 条。

注入到 system prompt 时使用固定结构：

```yaml
[User Preferences]
- reply.language: zh-CN (confidence=0.95, source=explicit)
- code.style: minimal-comments (confidence=0.70, source=inferred)
- task.default_timezone: Asia/Shanghai (confidence=0.90, source=explicit)
```

结构化比自然语言段落更稳定，模型不容易误读。

### 4. 更新机制：允许覆盖，但记录来源

写入时检查是否已有同名 key。如果旧值和新值不同，不要立刻静默覆盖。可以按来源处理：

- `explicit` 用户明确说出的偏好，优先覆盖。
- `inferred` 从行为推断出来的偏好，只提高 confidence，不直接覆盖显式偏好。
- `derived` 从其他偏好推导出来的，更新时保留溯源。

每条记忆最好记录：`source`、`confidence`、`last_updated`、`ttl`。

### 5. 存储选型

单用户本地使用 JSON 或 SQLite 足够。多用户或跨会话检索可以用 SQLite + FTS5。

路径建议放在用户目录下，例如：

```text
~/.openclaw/memory/preferences.json
~/.openclaw/memory/episodic.sqlite
```

不要一上来就接向量数据库，除非你已经有明确的语义检索需求。

## 踩坑点

### 1. 把“说过一次”当成长期偏好

用户在调试时随口说“这次用英文”，不应该立刻写进长期偏好。需要用 confidence 区分：显式表达高置信，行为推断低置信。低置信的记忆不要直接注入，或只在检索时降低权重。

### 2. 上下文污染

记忆注入会占用 context。每轮注入太多记忆，模型反而忽略关键偏好。控制 top-k，压缩记忆内容，不要保存整段对话原文。

### 3. 冲突处理

用户先说要英文回复，后来又用中文聊天，这不代表长期偏好变了。可能只是临时切换。可以记录“短期覆盖”，而不是马上改长期记忆。

### 4. 多身份隔离

如果 OpenClaw 接多用户或同时处理多个项目，必须加 `scope` / `user_id` / `project_id`。没有隔离的记忆系统基本不可用。

### 5. 敏感信息泄漏

API key、密码、token 不应该进入长期记忆。保存前做过滤，明确禁用 `password`、`secret`、`token` 等键名。

## 可复用建议

- 第一版只做 `save_preference` + `get_preference`，配合 system prompt 注入显式偏好。
- 偏好键名用命名空间，如 `reply.language`、`code.style`、`task.default_timezone`。
- 每条记忆必须带 `source`，显式优先，推断降权。
- 给偏好设置 TTL，临时偏好自动过期。
- 检索时用短查询，不要用整段对话当 query。
- 在 OpenClaw 调试时打开 memory tool 日志，确认每轮实际注入了哪些偏好。

## 总结

Agent 记忆系统的关键不是“记住更多”，而是“少而准”。把偏好结构化、按需检索、可过期、可溯源，才能真正让 AI 助手稳定地理解你的习惯。第一版先满足显式偏好 + 可控注入，再逐步迭代推断和语义检索。这样最务实，也最好维护。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/1627bd355e214a9f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/dc92e33640b45a1a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/bf1aec9526b6f0ac.png)

