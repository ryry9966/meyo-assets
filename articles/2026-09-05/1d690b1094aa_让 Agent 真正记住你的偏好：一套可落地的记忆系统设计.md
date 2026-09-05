---
title: 让 Agent 真正记住你的偏好：一套可落地的记忆系统设计
feedId: 36170
source: 综合讨论
publishedAt: 2026-09-05
---

## 背景

LLM 本身无状态，每次会话都是一张白纸。用过 Agent 做自动化的人都有体感：你得反复交代"commit message 用英文""项目用 pnpm 别用 npm""我时区是 UTC+8"。这些偏好不属于任何一次任务，却决定了每次任务的输出质量。记忆系统要做的，就是把这类信息从对话流里捞出来、持久化，并在合适的时机喂回去。

## 问题

三种常见做法都会失败：

- **全量塞上下文**：token 成本线性增长，信噪比暴跌，模型反而被无关历史带偏；
- **只靠手写 system prompt**：不可扩展，改一次要动一次配置，Agent 也无法自主更新；
- **会话摘要**：摘要把"一次性决定"和"长期偏好"混在一起，几天后膨胀成一锅粥。

核心矛盾一句话：**写入必须低噪声，读取必须确定性。**

## 做法

我们通过 MCP 拆出两个工具（`memory_search` / `memory_write`），底层 SQLite + 一张 metadata 表，分四步：

1. **存储分层**。每条记忆是结构化条目，不是自由文本。schema 至少包含：type（preference / fact / procedure）、scope（global / project）、value、confidence、source、created_at、last_used_at。偏好和事实分开存，因为生命周期完全不同。

2. **写入路径**。不在对话中实时写。会话结束后跑一个提取 pass：让模型从对话中抽取候选条目，按 schema 结构化，再做去重（embedding 相似度 + key 冲突检测）。置信度低于阈值或来源模糊的，丢弃而不是存。

3. **读取路径**。新会话启动时按 scope + 相关性检索：偏好类直接注入 system prompt（这一步要确定性，不要再过一次 LLM）；事实类作为可检索工具留给 Agent 按需调用。注入条目设 token 上限，超限按 last_used_at 截断。

4. **纠错回路**。用户说"我说过不用 npm"而记忆里没有，或者记忆本身就是错的，这个信号必须能触发更新和删除。给 `memory_write` 加 supersede 语义：新条目显式作废旧条目，不做盲目 merge。

## 踩坑点

- **过度提取**："这次用 Python"不是永久偏好。提取 prompt 里必须明确区分一次性和长期，宁可漏存让用户复述，不要错存以后天天错。
- **记忆腐化**：偏好会变。last_used_at 超过两个月且低频命中的条目标记 stale，下次命中时向用户确认一遍。
- **注入风险**：记忆值可能来源于网页抓取内容。存入前做清洗，读取注入时保持"这是用户数据"的定界，别让记忆变成持久化的 prompt injection 载体。
- **本地优先**：记忆存本地，用户可查看、可删除。这是底线，也省掉一堆合规麻烦。

## 可复用建议

- 先做 JSON 文件 + 关键词匹配的 MVP，验证提取质量后再上 embedding，别一上来就堆向量库；
- 写一个 10 条左右的回归测试集（"Agent 是否记得 X"），每次改提取 prompt 跑一遍；
- 注入预算固定（比如 300 token），倒逼自己把相关性排序做好。

## 总结

记忆系统本质是数据工程问题：分层的 schema、克制的写入、确定性的读取、可用的纠错回路，四者缺一不可。它不性感，但跑通之后，Agent 才能从"每次都要重新教"变成"越用越顺"。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/dc42d4a874a25483.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/3518e7fb91329cc8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/632b532fc4305bf5.png)

