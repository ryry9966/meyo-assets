---
title: Agent 记忆系统设计：让偏好真正落库，而不是塞进系统提示词
feedId: 34088
source: 综合讨论
publishedAt: 2026-08-22
---

很多 OpenClaw 用户会把“记住我喜欢简洁回答”直接写进 system prompt，或者每次对话开头手动贴一段个人偏好。结果 system prompt 越来越长，Agent 开始忽略工具定义；偏好一旦变化要手动改多处，跨会话也不一致。问题通常不在模型不够聪明，而在于把“记忆”当成了静态提示词。

真正可用的 Agent 记忆，至少要有四层：

- **L0 会话工作记忆**：当前任务目标、最近操作、中间结果，随会话结束清理。
- **L1 用户偏好档案**：语气、格式、工具选择、语言、隐私边界等稳定偏好。
- **L2 项目/领域上下文**：项目路径、依赖、部署环境、约定。
- **L3 决策/事件日志**：做过什么、为什么这样选，用于未来回溯。

把这四层分开，是为了区隔写入频率、检索方式和生命周期。L1 可以是结构化键值；L2 适合向量检索；L3 适合追加日志并按时间衰减。

## 做法与步骤

我不建议一上来就做全自动记忆。全自动记忆很容易把用户测试用的“以后都用英文回复”写成永久偏好。更稳的路径是：显式记忆优先，自动候选兜底。

### 1. 先建立最小 schema

以 SQLite 为例，偏好表可以这样设计：

```sql
CREATE TABLE preferences (
  id INTEGER PRIMARY KEY,
  user_id TEXT NOT NULL,
  scope TEXT NOT NULL,          -- global / project / session
  key TEXT NOT NULL,
  value TEXT NOT NULL,
  source TEXT NOT NULL,         -- explicit / auto_candidate / tool
  confidence REAL DEFAULT 0.5,
  created_at TEXT,
  updated_at TEXT,
  last_used_at TEXT,
  UNIQUE(user_id, scope, key)
);
```

这里 `scope` 很重要。不要把所有偏好都放 `global`，否则不同项目间的偏好会互相污染。

### 2. 把记忆做成 MCP Server

将记忆能力封装成 MCP 工具，例如 `search_memory`、`save_preference`、`review_conflicts`。OpenClaw 通过 MCP 调用，而不是把记忆逻辑写死在主循环里。这样做的好处是换模型、换 Agent 框架都能复用，而且可以单独做权限与审计。

### 3. 检索时分层召回

每次组装上下文不要全量注入。可以按这个顺序：

1. 注入最近会话摘要，只保留最近 3–5 轮关键动作；
2. 按 `scope` 匹配当前项目，取置信度大于 0.6 且最近使用过的 top 8 条偏好；
3. 对项目知识做向量检索，只取 top 3–5 个片段；
4. 如果本次任务涉及历史决策，再追加相关 L3 日志。

### 4. 写入与冲突处理

写入前先查同 key 是否已有值。如果已有，比较 `confidence`、`source`、更新时间。显式用户指令，例如“以后默认用中文”，其置信度应高于自动候选。若冲突无法自动判断，不要覆盖，而是标记为 `needs_review`，把候选值存进待审表或返回给用户确认。这可以避免“用户随口说一句就永久改偏好”的坑。

## 踩坑点

- **只写不读**：很多团队做了记忆存储，但检索时为了省 token 只取最新一条，等于没有长期记忆。
- **上下文污染**：把几十条偏好全量塞进 system prompt，模型会分不清主次，甚至影响工具调用。记忆注入应该是压缩过的、有选择的。
- **敏感信息**：不要把密钥、token、个人隐私直接写进通用记忆库。敏感字段建议单独加密，或使用本地 SQLite/向量库，不要默认上云。
- **冲突策略太简单**：很多实现用“最新覆盖”，但用户测试时说的话可能不是真实偏好。必须保留来源和置信度。
- **自动提取太激进**：不要一见到“以后”就写库。需要规则 + LLM 评分，比如同时满足“明确指向未来行为”“不是一次任务指令”“与已有偏好不冲突”才写入。

## 可复用建议

- 从显式命令开始：让用户说“记住：……”，先跑通存储、检索、更新闭环。
- 保留可见性：定期输出“我目前记住的偏好”，让用户能纠正。
- 把记忆系统当作独立 MCP 服务，而不是 Agent 内置功能。
- 记忆不是越多越好，按使用频率和时效衰减，长期不用自动降权或归档。

## 总结

Agent 记忆系统的核心不是“存得多”，而是“在该出现的时候出现，不该出现的时候安静”。分层、有来源、可回放、可用户纠正，才能让 AI 助手真正记住你的偏好，而不是把 system prompt 变成一个越来越慢的垃圾场。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/e976031a4c95c16c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/eb2a350c3dc45076.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/2cd40cec4d9f8e78.png)

