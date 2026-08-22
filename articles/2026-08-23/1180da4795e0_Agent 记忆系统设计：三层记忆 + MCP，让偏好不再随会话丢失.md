---
title: Agent 记忆系统设计：三层记忆 + MCP，让偏好不再随会话丢失
feedId: 34256
source: 综合讨论
publishedAt: 2026-08-23
---

# Agent 记忆系统设计：三层记忆 + MCP，让偏好不再随会话丢失

## 背景：Agent 默认是“失忆”的

在 OpenClaw 里接 MCP、插件、自动化脚本后，最常遇到的情况是：模型在某个会话里已经知道你说“不要输出 emoji”“代码注释用英文”“PR 描述用中文”，但新开会话后又开始按照默认风格回答。根因不是模型能力，而是我们没给它设计记忆落盘与检索路径。

很多实践者第一反应是把聊天记录丢进向量库。但聊天记录是非结构化、含大量噪声的，如果直接当长期记忆，检索回来的经常是无关片段，还会挤占上下文窗口。更实用的做法是先分类，再决定存储和注入策略。

## 问题拆解

Agent 记忆要解决四个问题：

1. 记什么：哪些偏好值得长期保留，哪些只是一次性指令。
2. 存哪里：文件、SQLite，还是向量库。
3. 怎么取：新会话上下文里注入多少记忆、按什么顺序。
4. 怎么失效：偏好过期、冲突、用户反悔时如何处理。

如果只“写”不“管”，记忆系统会从有用变成污染源。

## 做法：三层记忆 + MCP 读写接口

### 1. 先做三层分类，不要一上来搜全局

把记忆分成三类：

- **L1 工作记忆**：当前任务里的临时变量，例如“本次批量重命名要跳过 test 目录”。只存会话内或短期文件，任务结束即清理。
- **L2 用户画像**：稳定偏好，例如“回复语言中文”“代码风格使用 4 空格缩进”“解释概念时先给结论再展开”。这类是长期记忆主体。
- **L3 项目上下文**：和某个仓库/自动化流程相关的约定，例如“这个项目 CI 必须在 Node 18 跑”“发布前必须人工确认版本号”。可绑定路径或项目名。

这样做的好处是检索范围可控。新会话开始时，不会把 L1 的一次性指令带进去。

### 2. 小规模用 SQLite，比向量库更稳

如果记忆量在几百条以内，SQLite 完全够用。结构建议：

```sql
CREATE TABLE memory (
  key TEXT PRIMARY KEY,
  value TEXT NOT NULL,
  scope TEXT NOT NULL,          -- user / project / task
  source TEXT,                  -- user_instruction / tool_output / model_candidate
  confidence REAL DEFAULT 0.5,
  created_at TEXT,
  last_accessed_at TEXT,
  expires_at TEXT
);
```

需要语义匹配时，再考虑 sqlite-vec 或本地向量库。不要为了“看起来高级”直接上向量检索，早期更多是 FTS 或前缀匹配就能覆盖偏好查找。

### 3. 通过 MCP 暴露三个最小工具

在 OpenClaw 里，建议把记忆能力做成一个 MCP server，而不是由某个插件自己读写文件。这样所有插件和 Agent 共用一套记忆接口，避免各自维护一份。

最小工具集：

- `memory.remember(key, value, scope, ttl_days)`：写入或更新记忆。
- `memory.search(query, scope=None)`：按关键词/语义检索，返回 top-k。
- `memory.forget(key)`：删除记忆。

MCP server 可以只封装 SQLite，几十行就能跑起来。模型在任务结束前可以生成一条 `memory.remember` 调用，但需要经过过滤规则再落库。

### 4. 注入策略：少量、相关、带时间戳

新会话组装 system prompt 时，不要全量注入长期记忆。只取当前任务最相关的 top 3~5 条，并附上时间信息：

```text
[user preferences]
- 解释概念时先给结论再展开 (confidence: 0.9, updated: 2025-01-10)
- 代码注释使用英文 (confidence: 0.8, updated: 2025-01-12)
```

同时在系统提示中加一句：“以上偏好可能过时，若与用户当前指令冲突，以当前指令为准。”这能降低旧偏好干扰新任务的风险。

## 踩坑点

- **一次性指令写入长期记忆**：比如“这次回复用繁体中文”被记住后，下周还在用繁体。解决：默认只把用户重复出现或明确说“以后都这样”的指令写入 L2。
- **冲突偏好无裁决**：同一条 key 反复写，旧值被覆盖，但新值可能是低置信度候选。写入前比较 confidence，高置信度才覆盖低置信度，否则保留或标记待确认。
- **全量注入挤爆上下文**：200 条偏好直接塞进 system prompt，新任务输入反而放不下。务必用检索返回 top-k，并限制总字符数。
- **没有过期和删除**：用户改了偏好，但旧记忆还在。必须支持 `forget` 和 `expires_at`。
- **敏感信息落盘**：API key、token、服务器密码不要写进 memory 表。应使用 OpenClaw 的环境变量注入或专门 secret store。

## 可复用建议

1. 每条记忆必须带 `source`、`confidence`、`created_at`、`expires_at`。
2. 给用户可手动管理的命令，比如在对话里说“记住：以后代码注释用英文”“忘记我之前关于注释的偏好”。这比完全自动写更可控。
3. 检索加阈值，低于阈值不注入；同时记录 `last_accessed_at`，长期未命中的低置信度记忆定期清理。
4. 项目上下文绑定路径或项目名，切换项目时只加载对应范围的记忆。
5. 先用文件或 SQLite 跑通，再决定是否上向量库。大多数 Agent 偏好记忆的瓶颈不在检索算法，而在写入质量和失效机制。

## 总结

Agent 记忆系统不是“把聊天记录存起来”，而是把高频、稳定、可验证的用户偏好变成可检索、可更新、可删除的结构化数据。三层分类控制范围，SQLite/MCP 控制工程复杂度，注入和过期策略控制上下文预算。跑通这套后，AI 助手“记得住偏好”才会变成稳定的体验，而不是偶尔灵光一现。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/b346dfd3dac3db46.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/23de293b412f1cd6.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/4d63f7d8a6b473cc.png)

