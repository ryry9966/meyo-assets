---
title: Agent 记忆系统设计：从无状态到真正记住你的偏好
feedId: 33637
source: 综合讨论
publishedAt: 2026-08-17
---

## 背景

很多 Agent（包括 OpenClaw 工作流）每次会话都像失忆：用户说过“日志格式用 JSON”“部署环境是公司内网 k8s”“代码风格用 TypeScript”，下个会话还要重新问一遍。原因是底层 LLM API 无状态，而我们往往只把最近几条消息塞进上下文，没有真正的记忆层。

直接把全部历史对话塞进 system prompt 也不现实：上下文窗口有限，历史里大量噪音反而会稀释指令。我们需要区分“短期工作状态”和“长期偏好”，并且让长期偏好可存储、可检索、可更新。

## 问题

记忆系统至少要解决三件事：

1. **采集**：从对话中提取值得长期保存的偏好，而不是把临时需求也记下来。
2. **存储与检索**：偏好不能只存 JSON 文件，需要考虑冲突、过期、多项目隔离。
3. **注入**：在合适的时间把相关偏好放进 system prompt，且不挤占上下文。

在 OpenClaw 里，我通常把记忆能力拆成一个 MCP server，暴露 `save_preference`、`search_preferences`、`forget_preference` 三个工具。Agent 在对话中调用这些工具，而不是直接读写数据库。

## 做法/步骤

### 1. 定义偏好模型

不要用裸 key-value。每条偏好至少包含：

```yaml
- preference_id: "uuid"
  namespace: "user:alice"
  domain: "code_style"
  key: "language"
  value: "TypeScript"
  confidence: 0.9
  source: "conversation"
  ttl: null
  updated_at: "2025-03-10T10:00:00Z"
```

`namespace` 用来区分用户或项目；`domain` 用来防止同名字段串味，比如 `deploy.env` 和 `dev.env` 是两个域。

### 2. 采集与提取

在 Agent 处理完回复后，加一个“记忆提取”步骤。用 LLM function call 输出偏好条目，但要求它只提取明确长期意图的信号，例如：

- “以后都用 TypeScript”
- “我习惯日志输出 JSON”
- “不要在生产环境自动重启”

临时需求如“这次先用 Python 试试”不能被记成长期偏好。可以通过 prompt 约束和 `confidence` 分数控制：只有 signal 明显时 confidence 才高，否则丢弃。

### 3. 存储

结构化偏好放 SQLite，便于查询和审计；对话摘要或较长的偏好上下文放向量库，做语义召回。MCP memory server 内部封装这两层。

SQLite 表大致如下：

```sql
CREATE TABLE preferences (
  id TEXT PRIMARY KEY,
  namespace TEXT NOT NULL,
  domain TEXT NOT NULL,
  key TEXT NOT NULL,
  value TEXT NOT NULL,
  confidence REAL DEFAULT 0.8,
  source TEXT DEFAULT 'conversation',
  ttl INTEGER,
  updated_at TEXT NOT NULL,
  UNIQUE(namespace, domain, key)
);
```

### 4. 检索与注入

每次会话开始，Agent 先调用 `search_preferences`，根据当前任务做两路召回：

- 精确匹配：按 `namespace + domain + key` 查结构化偏好；
- 语义召回：用当前任务 query 在向量库里找相关偏好。

然后按 `confidence` 排序，设置 token 预算，比如最多 800 tokens，注入 system prompt 的 `user_preferences` 区块。每条偏好附上来源和更新时间，方便追溯。

### 5. 更新与冲突

当同一 `namespace + domain + key` 有新的偏好进入，比较 `confidence` 和 `updated_at`，新的高置信度偏好覆盖旧值，并保留历史记录。`ttl` 到期的偏好自动清理。用户也可以显式说“忘掉这个偏好”，调用 `forget_preference`。

## 踩坑点

- **把临时需求当长期偏好**：这是最常见的问题。解决方法是设计提取 prompt 时强调信号词，同时给偏好评 `confidence`，低于阈值的只存为短期摘要，不进入长期库。
- **检索过度**：一次注入太多不相关偏好，会让 system prompt 变长且淹没主指令。务必设置相关度阈值和 token 预算。
- **键名冲突**：不同项目都用 `env`，导致 `localhost` 和 `生产 k8s` 混淆。必须用 `domain` 做隔离。
- **隐私泄露**：偏好可能包含 API key、内部域名、路径。建议本地存储、加密，并记录访问日志。不要默认上传到云。
- **多 Agent 串味**：多个 Agent 共用一个记忆库时，一定要有 `namespace`，否则 A 项目的偏好会污染 B 项目。

## 可复用建议

- 从最小闭环开始：一个 SQLite 表 + 一个 `search_preferences` MCP 工具，先解决“用户明确说过的偏好能召回”，再考虑语义检索和自动总结。
- 给每条偏好加上 `source` 和 `confidence`，方便调试。你可以在 OpenClaw 的 trace 里看到哪条偏好被注入，却根本没被使用。
- 记忆系统的可观测性比模型本身更重要。记录命中/未命中的偏好，定期清理低置信度、长期未命中的条目。
- 长期记忆可以先走“规则 + LLM 提取”，不要一上来就做复杂的自主记忆 Agent。可控性比智能感更重要。

## 总结

Agent 记忆不是存得越多越好，而是存得对、检索准、注入可控。无状态 LLM API 之上需要一层显式的记忆中间层，MCP 是合适的接口。把偏好设计成可版本、可解释、可撤销的配置，而不是黑箱状态，这样才能真正让 AI 助手“记住”你的习惯，而不是每次都从头猜。

---

