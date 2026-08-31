---
title: Agent 记忆系统设计：怎么让 AI 助手真正记住你的偏好
feedId: 35602
source: 综合讨论
publishedAt: 2026-09-01
---

## 背景

OpenClaw 这类 Agent 框架默认把每次对话当作独立上下文，即使有会话历史，重新启动后也记不住你的偏好。很多人第一反应是“让模型自己总结并写进一个文件”，但做出来以后发现要么上下文越来越臃肿，要么旧偏好改不掉，最后只能删库重来。

问题不在“有没有记忆”，而在“记忆如何进入上下文、如何更新、如何过期”。一个能落地的偏好记忆系统需要解决三件事：存什么、什么时候取、冲突时听谁的。

## 做法/步骤

我目前的方案是本地 SQLite + MCP 工具，不依赖任何外部记忆服务。核心设计分四步。

### 1. 定义最小记忆 schema

不要用自由文本当记忆。每条偏好至少包含：`key`（唯一标识）、`category`（工具偏好/格式偏好/事实/禁区）、`value`、`priority`（1-5）、`source`（`user_explicit` / `agent_inferred`）、`updated_at`、`expires_at`。

```sql
CREATE TABLE memory (
  key TEXT PRIMARY KEY,
  category TEXT NOT NULL,
  value TEXT NOT NULL,
  priority INTEGER DEFAULT 2,
  source TEXT DEFAULT 'agent_inferred',
  updated_at INTEGER NOT NULL,
  expires_at INTEGER
);
```

SQLite 方便加 TTL 和索引，JSON 文件也能用，但并发写入和过期清理不如 SQLite 顺手。

### 2. 通过 MCP 暴露三个工具

在 OpenClaw 里，把记忆做成 MCP server，暴露：

- `remember_preference(key, value, category, priority, ttl_days)`
- `search_memory(query, category, limit)`
- `forget_preference(key)`

这样 Agent 可以按需读取或写入，而不是每次把整个记忆库塞进 prompt。MCP 的好处是记忆模块和主 Agent 解耦，其他插件或自动化流程也能复用同一套记忆。

### 3. 注入策略：少量高优 + 按需检索

系统提示只注入 `priority >= 4` 且未过期的偏好，最多 8-10 条。例如“你回复默认使用中文，代码风格用 4 空格缩进”。低优先级的内容不预加载，由 Agent 在任务中遇到工具选择或格式要求时调用 `search_memory` 查询。这样上下文消耗可控，模型也不会被大量旧偏好干扰。

### 4. 更新与冲突处理

用户显式说“以后都按这个来”时，写 `priority=5` 且无过期时间；Agent 从行为中推测的偏好写 `priority=2` 且 7 天过期。遇到同 key 不同 value 时，不静默覆盖，而是保留旧值并提醒用户确认。我们踩过坑：隐式学习太积极，把一次性的临时选择记成长期偏好，结果用户下次非常恼火。

## 踩坑点

- **全量注入记忆会污染上下文**，模型容易把旧偏好当成硬规则，甚至拒绝用户的新指令。
- **隐式学习必须有衰减**，否则会不断积累噪声，一周后记忆库里全是过时的推测。
- **冲突不要自动解决**，用户确认成本远低于纠错成本。宁可多问一次，也别猜错方向。
- **敏感信息不要写进普通记忆表**。API key、地址、账号密码这类东西要么加密存储，要么干脆不记。
- **记忆格式如果靠模型自由生成，会越来越乱**。一定要用 schema 约束 key 和 value 的结构，不要存大段自然语言。

## 可复用建议

把记忆分成三层：

1. **会话记忆**：当前任务上下文，不进库。
2. **工作记忆**：最近 N 天内的临时偏好，设置 TTL。
3. **长期偏好**：用户显式确认、几乎没有过期时间的规则。

默认“先问再记”：模型想记住用户偏好时，先向用户确认或标记为低置信度。导出和清空功能必须有，便于用户审计和重置。在 OpenClaw 里优先用 MCP 暴露记忆，而不是写死在自定义指令里，这样后续换模型或加插件，记忆层都能稳定工作。

## 总结

让 Agent 记住偏好，本质是管理上下文预算和信任。好的记忆系统不是存得越多越好，而是在正确时机把少量正确信息放进 prompt，并且允许用户随时修改或遗忘。工程上，一个 SQLite + 三个 MCP 工具 + 分层注入策略，就能覆盖多数个人自动化场景，足够小，也足够可靠。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/b4996bf544014e8a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/e59d1d14d44f0256.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/f0a0361acfc20497.png)

