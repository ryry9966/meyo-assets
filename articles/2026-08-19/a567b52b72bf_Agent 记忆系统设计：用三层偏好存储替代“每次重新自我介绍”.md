---
title: Agent 记忆系统设计：用三层偏好存储替代“每次重新自我介绍”
feedId: 33837
source: 综合讨论
publishedAt: 2026-08-19
---

## 背景

接入 OpenClaw、MCP 或自动化插件后，很多 Agent 的规划、工具调用和代码生成能力都不差，但每次新开会话仍然要重新问一遍：输出用中文还是英文？代码优先 TypeScript 还是 Python？部署前要不要先 dry-run？

这不是模型能力问题，而是记忆没有被结构化沉淀。仅依赖聊天历史，上下文窗口会快速膨胀，偏好散落在自然语言里，既不稳定读取，也很难更新。

## 问题拆解

一个可用的 Agent 记忆系统至少要解决三件事：

1. 区分短期状态和长期偏好，不能把会话里的临时决定当成长久规则。
2. 偏好需要可写入、可更新、可遗忘、可审计，而不是只塞进 system prompt。
3. 最好通过 MCP 工具暴露给 Agent，让记忆读写成为显式动作。

如果你的 Agent 只能靠“记住用户说过什么”来维持偏好，那基本可以认为它没有长期记忆。

## 做法：先做一个本地 SQLite 版 memory MCP server

不建议一上来就上向量库或知识图谱。对偏好类记忆来说，结构化的 key/value 存储更可控。

### 1. 分层模型

- **短期会话记忆**：当前任务状态、临时决定，会话结束即可丢弃。
- **长期偏好**：跨会话稳定的用户选择，例如“回答用中文”“部署前必须 dry-run”“日志默认 JSON 格式”。
- **事实/规则**：项目环境信息，如“项目 A 使用 PostgreSQL”“本地 MCP 端口 8765”。

第一层可以先不做持久化，重点放在第二层。长期偏好是最容易见效的部分。

### 2. 存储结构

用 SQLite 足够，表结构如下：

```sql
CREATE TABLE preferences (
  key TEXT PRIMARY KEY,
  value TEXT NOT NULL,
  scope TEXT NOT NULL DEFAULT 'user', -- user/project/agent
  confidence REAL NOT NULL DEFAULT 0.6,
  source TEXT NOT NULL DEFAULT 'inferred', -- inferred/explicit
  namespace TEXT NOT NULL DEFAULT 'general',
  updated_at TEXT NOT NULL,
  expires_at TEXT
);
```

key 使用命名空间，例如：

- `communication.language`
- `tooling.preferred_shell`
- `output.code_fence`
- `deployment.require_dry_run`

不要设计成“记住任意文本”，否则很快会变成新的垃圾堆。

### 3. MCP 工具接口

暴露 4 个工具给 Agent 调用：

- `remember_preference(key, value, scope, namespace, confidence, expires_at)`
- `get_preferences(namespace?, scope?, min_confidence?)`
- `update_preference(key, value, confidence_delta?)`
- `forget_preference(key)`

工具名字可以更短，但功能要保持这四类：写入、读取、更新、删除。

### 4. 写入策略

不要让 Agent 每句话都尝试写偏好。写入至少满足以下一个条件：

- 用户显式声明：“以后都用 UTC 时间”。
- 同一偏好跨 2 个及以上会话重复出现。
- Agent 执行时因缺失偏好导致返工，事后补记。

默认 confidence 从 0.6 开始。同一 key 再次确认时 +0.2，上限 0.95。冲突时相同来源以最新值为准，不同来源给 `explicit` 更高权重。

### 5. 读取策略

新会话启动时，只注入与当前任务相关的偏好。可以先用任务关键词匹配 namespace，而不是全量注入。

例如用户说“帮我写部署脚本”，只读取 `scope=project` 且 namespace 为 `tooling`、`deployment`、`output` 的偏好，拼接成紧凑 JSON 放进系统提示或第一个工具返回里。不要让偏好占用超过 800–1200 tokens。

## 踩坑点

- **什么都记**：如果提示词写“观察用户喜好并记住它”，Agent 很可能记下“用户今天心情一般”这类噪音。偏好必须稳定、可操作。
- **敏感信息入库**：密码、token、内网地址不要进入 preferences 表。如果必须存，走本地 secret store 或加密字段，并设置 `expires_at`。
- **只增不改**：用户改主意后，旧偏好仍以高 confidence 存在。必须提供 update 和 forget，并保留 audit log，方便排查“为什么 Agent 还在用旧规则”。
- **上下文污染**：把全量偏好塞进 system prompt 会挤占工具调用空间，尤其在使用多个 MCP 工具时，容易影响指令遵循效果。

## 可复用建议

- 先做本地 SQLite 版 memory MCP server，跑通后再考虑同步或更复杂存储。
- 命名空间提前规划：`communication`、`tooling`、`output`、`domain`、`experiment`。
- 给用户一个 `/memory` 或 `!prefs` 命令，用于导出、清空、删除偏好。
- 设置置信度阈值，只有 `confidence >= 0.7` 的偏好才自动注入；低置信度内容先询问用户确认。
- 为偏好加 TTL：临时偏好 7 天，项目偏好 90 天，用户级长期偏好可设 365 天或永久，但永久项需要用户确认。

## 总结

Agent 记忆不需要复杂架构。把偏好结构化成 key/value，通过 MCP 工具写入 SQLite，按置信度和命名空间做注入，再补上更新、遗忘、审计三件事，就能从“每次重新自我介绍”变成真正可用的长期助手。

先让偏好能被稳定读写和删除，再谈更高级的记忆检索。

---

