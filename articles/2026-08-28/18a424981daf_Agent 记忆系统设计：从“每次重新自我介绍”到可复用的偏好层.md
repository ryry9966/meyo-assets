---
title: Agent 记忆系统设计：从“每次重新自我介绍”到可复用的偏好层
feedId: 34992
source: 综合讨论
publishedAt: 2026-08-28
---

## 背景

OpenClaw 里接入 MCP 工具后，Agent 能查资料、跑命令、调插件，但很多实践者发现一个尴尬问题：每次新会话，AI 就像失忆一样，你反复强调“不要用 docker compose down 清数据”“日志用中文”“代码风格遵循 ruff”，下一次还得重新说。把偏好堆进 system prompt 又会让上下文越来越长，且难以维护。

因此需要一个独立的记忆系统，把“偏好”从主 prompt 中拆出来，按需读写。

## 问题

设计记忆系统时，常见问题不是“要不要记”，而是：

- 记在哪里？
- 什么时候写？
- 怎么检索？
- 怎么避免记错、记脏、记冲突？

下面给一个适合 OpenClaw + MCP 的最小实现。

## 做法/步骤

### 1. 记忆分层

建议至少分三层：

- 短期记忆：当前会话上下文，由 OpenClaw 自己维护，不落盘。
- 工作记忆：近期项目相关的事实，比如“当前项目路径”“本次任务目标”“临时禁用某个工具”。
- 长期记忆：稳定偏好，比如代码风格、响应语言、工具偏好、称谓、明确禁忌。

长期记忆是偏好系统的核心，应该结构化存储，而不是一段自然语言。

### 2. 结构化存储

用一个 JSON 文件或 SQLite 即可起步。示例：

```json
{
  "user": "default",
  "preferences": {
    "code_style": {
      "language": "python",
      "formatter": "ruff",
      "typing": "strict"
    },
    "response_style": {
      "language": "zh-CN",
      "tone": "concise",
      "format": "markdown"
    },
    "tooling": {
      "preferred_shell": "fish",
      "preferred_editor": "nvim"
    }
  },
  "updated_at": "2026-03-20T10:00:00Z"
}
```

每条偏好建议包含 `key`、`value`、`scope`、`source`、`timestamp`、`confidence`。字段不用复杂，但来源和时间一定要有，否则无法审计。

### 3. 用 MCP 暴露记忆工具

写一个 memory MCP server，提供几个工具即可：

- `remember(key, value, scope, source)`
- `recall(query_or_scope, limit)`
- `update_preference(key, value, source)`
- `forget(key)`

OpenClaw 侧注册 MCP server 后，在 agent 指令中增加规则：

- 当用户明确表达长期偏好或纠正你的行为时，调用 `remember` 写入。
- 当任务涉及代码风格、响应格式、工具选择时，先 `recall` 再回答。
- 不确定用户是否希望长期记住时，不要写；可以先问。

这样主上下文只保留规则和少量检索结果，记忆数据在外部，避免 token 爆炸。

### 4. 写入与检索策略

写入触发条件要严格。用户说“以后都用 uv 而不是 pip”可以写；用户只说“这次用 pip”就不要写。给工具加 `confidence` 字段，用户明确表达写 `confirmed`，Agent 推测写 `unconfirmed`。

检索可以用最简单的 `scope` 过滤，例如 `recall("code_style")` 返回整个代码风格偏好。条目多了以后，再用 SQLite FTS5 或 embedding 做语义检索。个人助手规模通常几百条以内，全量加载加内存过滤完全够用。

### 5. 更新与冲突处理

同 key 更新时保留旧值，例如：

```json
{
  "key": "code_style.formatter",
  "value": "ruff",
  "previous_value": "black",
  "source": "user",
  "updated_at": "..."
}
```

冲突时以最近的、明确表达的用户指令为准。工具可以做成 diff 模式，避免模型直接覆盖整个 JSON 文件。

## 踩坑点

- 把所有偏好塞进 system prompt：几轮之后上下文就废了。
- 用自然语言段落存偏好：看似灵活，实际检索不到，也容易冲突。
- 把一次性选择当长期偏好：导致记忆库迅速变脏。
- 模型自动写入垃圾数据：必须限制写入条件，必要时要求用户确认。
- 明文存储敏感信息：个人 Token、路径、账户信息尽量放环境变量或加密存储。
- 没有来源和时间：出问题无法回滚，也不知道是哪次对话写入的。
- 频繁写入导致 MCP 调用过多：可以批量或在会话结束时统一写入。

## 可复用建议

- 最小可用版本：一个 JSON 文件 + 一个 MCP server，四个工具足够。
- 偏好分类命名空间：`code_style`、`response_style`、`tooling`、`personal`，避免全局乱放。
- 写入必须带 `scope` 和 `source`，更新保留 `previous_value`。
- 检索优先精确 `scope`，不要全文搜索。
- 在 OpenClaw agent 指令中明确“一次性要求不写入长期记忆”。
- 定期检查记忆文件，删除过时条目；可以给条目加 `expires_at`。
- 隐私信息本地存储，不上传。

## 总结

Agent 记忆系统的重点不是“记住更多”，而是让记忆可检索、可更新、可审计。把记忆拆成外部 MCP 工具后，主上下文保持干净，偏好也不会在每轮对话中反复复制。先从结构化 JSON + 基础 MCP 工具做起，等条目多了再引入向量检索和过期策略，比一上来就上复杂框架更实用。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/09cc8b688bf15567.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/c707af5a35d0df15.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/5bf9f6c70e052b83.png)

