---
title: Agent 记忆系统设计：用 MCP 把用户偏好落成可检索的长期记忆
feedId: 33398
source: 综合讨论
publishedAt: 2026-08-16
---

## 背景

Agent 在自动化任务里最大的问题不是工具不够，而是每次运行都像第一次见面。用户说过“不要用 root 执行”“通知走飞书，不要邮件”“部署失败自动回滚”，但下一次任务依然忘。仅靠 system prompt 硬编码偏好，既不可扩展，也会撑爆上下文窗口。

## 问题

长期记忆需要解决三件事：存哪里、什么时候写、怎么取。  
如果每次把全部用户偏好塞进上下文，会浪费 token 并干扰推理；如果让 Agent 自动提取隐式偏好，又容易把一次偶然行为当成稳定偏好。

## 做法

### 1. 定义最小记忆 schema

用半结构化 JSON 存储，字段不要一开始贪多：

```json
{
  "id": "mem_01",
  "user_id": "default",
  "category": "notification",
  "key": "preferred_channel",
  "value": "feishu",
  "confidence": 0.9,
  "source": "user_explicit",
  "updated_at": "2025-01-01T00:00:00Z"
}
```

`category + key + value` 是核心，`confidence` 和 `source` 用于控制写入与更新策略。

### 2. 用 SQLite 存储，通过 MCP 暴露工具

建议使用 SQLite 单文件，开启 WAL，并给 `user_id` 建索引。通过 MCP server 暴露三个工具：

- `memory_search(query, user_id, limit=5)`
- `memory_upsert(user_id, category, key, value, confidence, source)`
- `memory_forget(memory_id, user_id)`

OpenClaw 配置好 MCP 后，Agent 可以直接调用这些工具，不用自己实现存储逻辑。

### 3. 写入策略：显式优先，隐式克制

- 显式写入：用户说“记住我以后部署都不加 --force”，Agent 调用 `memory_upsert` 写入，`source=user_explicit`，`confidence` 设高。
- 隐式写入：任务完成后，从对话摘要中抽取候选偏好，例如检测到用户两次拒绝某操作。但不要全自动写入，先进入待确认状态，或设置 `confidence` 阈值，低于阈值的必须由用户确认。

### 4. 检索与注入

每次新任务开始时，用当前任务描述或用户首条消息作为 query，调用 `memory_search` 返回 top 3-5 条，格式化后注入 system prompt：

```text
[User preferences]
- notification: preferred_channel=feishu
- deploy: allow_auto_rollback=true
```

超过 5 条截断，并标注 `updated_at`，避免使用过期偏好。

### 5. 更新与冲突

同 key 更新时，把旧值写入 history 字段或单独审计表，方便回滚。不要简单覆盖。若新来源 `confidence` 低于旧值，可跳过更新或要求确认。

## 踩坑点

- **长期记忆和短期上下文混在一起**：检索和清理都会变乱。建议长期记忆独立成表，短期上下文只保留当前任务过程。
- **盲目上向量库**：多数用户偏好是结构化键值，先用 `category + key` 精确匹配，配合 SQL 的 `LIKE` 就能覆盖大部分场景，向量检索收益有限。
- **隐式提取太激进**：有 Agent 把用户一次手动执行命令当成长期偏好，后续全自动执行，差点误删。必须设 `confidence` 门槛，低置信度写入需确认。
- **多用户/多 Agent 串号**：`user_id` 必须作为 MCP 工具必填参数，并在服务端校验，否则会互相污染。
- **JSON 文件并发写入**：多个 Agent 同时写会出现覆盖。SQLite 事务最省事；如果坚持 JSON，加文件锁或单写者模式。

## 可复用建议

1. 从显式记忆开始：用户说“记住”才写，运行一周后再考虑隐式抽取。
2. 工具命名固定：`memory_search` / `memory_upsert` / `memory_forget`，Agent 调用更稳定。
3. 每次只注入 top 3-5 条，每条记忆带 `source` 和 `updated_at`。
4. 必须提供“忘记”入口，否则用户无法清除错误偏好。
5. 定期审计低 `confidence` 或长期未更新的记录，降级或删除。

## 总结

Agent 记忆系统不需要复杂架构。用 SQLite + MCP 三个工具，配合显式写入和克制检索，就能让自动化任务真正记住关键偏好。先把 schema 和写入策略跑稳，再考虑隐式记忆和高级检索。

---

