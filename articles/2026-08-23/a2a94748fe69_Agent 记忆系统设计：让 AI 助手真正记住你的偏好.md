---
title: Agent 记忆系统设计：让 AI 助手真正记住你的偏好
feedId: 34276
source: 综合讨论
publishedAt: 2026-08-23
---

## 背景

在 OpenClaw 这类 Agent 编排里，很多人的做法是不断往 system prompt 里追加“我喜欢...”“不要...”。一开始有效，但随着插件和 MCP 工具接入，system prompt 越来越长，模型开始忽略长尾指令，跨会话也无法复用。记忆不是缺，而是乱。

## 问题拆解

一个能用的记忆系统至少要回答四件事：

1. 什么值得写进长期记忆？
2. 什么时候写？
3. 怎么存储和检索？
4. 冲突偏好如何更新？

如果没有这些规则，记忆就只是另一个聊天记录存储。

## 做法/步骤

### 1. 先做记忆分类

建议从三类开始：

- `profile`：用户长期偏好，如“默认 Python 3.12”“代码风格用 ruff”
- `working`：当前任务上下文，会话内有效
- `episodic`：任务过程记录，如“上次部署卡在 Nginx 权限”

`working` 不落盘，`profile` 和 `episodic` 进 memory server。

### 2. 写入策略

不要放任 Agent 自己决定写记忆，至少要有两层约束：

- 显式写入：用户说“记住...”“以后默认...”，直接调用 `memory.set`
- 隐式提取：异步任务从会话末尾提取偏好，必须带 confidence 阈值和 source

示例记录：

```json
{
  "scope": "user",
  "domain": "code_review",
  "key": "diff_size",
  "value": "prefer smaller diffs under 200 lines",
  "confidence": 0.82,
  "source": "session_20260112",
  "updated_at": "2026-01-12T10:20:00Z"
}
```

所有写入记录保留 source，方便回溯和删改。

### 3. 存储实现

先不上向量库。SQLite 或 JSONL 足够：

表结构：`id, scope, domain, key, value, confidence, source, updated_at, ttl`。

需要模糊匹配时，给 value 和 key 做 embedding 索引，优先用结构化过滤 `domain/scope`，再做 top-k 检索。这样不会因为语义相似把隔壁项目的偏好注入当前任务。

### 4. 检索注入

不要全量注入。会话开始时：

- 根据当前 domain 拉取 profile
- 只注入 top 5-8 条高置信度偏好
- 工具调用前如果涉及特定 domain，按 domain 查询
- 给注入内容加固定段落，比如 `## User Preferences (from memory)`

### 5. 更新和冲突

- 同一 key 再次写入时，新值提高 confidence 并更新 updated_at
- 用户明确说“以后不要...”应覆盖旧值，而不是叠加
- 长期未使用的偏好降低权重或过期删除
- 冲突规则：最近明确表达 > 高置信度旧记录

## 踩坑点

- **全量记忆塞进 system prompt**：窗口消耗快，模型会选择性忽略中后段指令。
- **把“用户可能喜欢”当成偏好**：要 confidence >= 0.75 才写入，否则只是候选。
- **没有作用域隔离**：个人偏好和项目规范混在一起，检索结果互相污染。
- **只靠向量检索**：结构化过滤缺失，导致错误记忆注入。
- **每次会话结束都做总结写入**：产生重复记录和冲突，应该查重和更新。
- **忽略隐私**：偏好可能包含路径、账号、部署信息，不要默认上传到第三方向量服务。

## 可复用建议

- 使用 MCP memory server 单独管理，工具至少有 `get/set/list/delete/search`
- 从 profile 偏好开始，不要一上来做复杂知识图谱
- 给记忆加 TTL 或访问计数，长期不用自动降权
- 记录记忆命中率：注入后用户是否纠正、工具结果是否匹配，用日志反推阈值
- 重要偏好写入后让 Agent 回执确认，避免静默覆盖

## 总结

Agent 记忆系统的重点不是“存更多”，而是让写入有阈值、存储有结构、检索有过滤、更新有冲突规则。先在 OpenClaw/MCP 里跑通一条偏好链路，比如“记住我的部署目录”到下次自动使用，比上向量库和知识图谱更有价值。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/9aede85cdfacea1c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/e8b4dfb54f378802.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/8af2a3bc1b586bdd.png)

