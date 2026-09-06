---
title: Agent 记忆系统设计：让 AI 助手真正记住你的偏好
feedId: 36334
source: 综合讨论
publishedAt: 2026-09-06
---

## 背景

LLM 会话是无状态的。每次新开会话，助手都忘了你偏好 pnpm、喜欢先看 diff 再动手、commit message 不写 scope 这些事，只能靠你重新交代一遍。多数人的第一反应是把偏好塞进 system prompt 写张"关于我"卡片，这能解决 20% 的问题，但很快会撞上上下文膨胀和记忆失效。

## 问题

我实际踩过的三个坑：

1. **全量注入不可持续**：偏好越攒越多，常驻 prompt 从 200 token 涨到 3000，注意力被稀释，模型反而开始忽略其中一半。
2. **摘要式记忆丢细节**：让模型每轮自总结存档，回头看时"用户偏好简洁风格"这种模糊句子还在，但硬约束早就没了。
3. **检索时序错乱**：三个月前说过"部署用 Docker"，上周改成了 Podman，向量检索把两条都召回，旧条目的相似度看起来还更高。

本质问题：记忆不是"存"的问题，而是写入过滤、结构化、更新与遗忘的工程问题。

## 做法

我用 MCP 实现了一个 memory server（SQLite + sqlite-vec），挂到 OpenClaw 的 Agent 上。核心是四层结构加两条路径。

**四层结构：**

- **Profile（常驻）**：固定 key 的结构化偏好卡，如 code_style、timezone、response_language，每条 value 带更新时间。
- **Preference（长期）**：非固定 key 的好恶，必须包含负向偏好（"不要用 emoji"）。
- **Episode（情景）**：会话级摘要，供检索。
- **Procedure（流程）**：沉淀下来的操作步骤，如"发布前跑 lint + 单测"。

**写入路径**：每轮结束跑一次轻量抽取，产出候选记忆 `{type, key, content, confidence, source, ts}`，过三道闸：置信度低于 0.7 丢弃；与现有条目 embedding 相似度超 0.92 视为重复；同 key 冲突时新覆盖旧并保留历史版本。每会话写入上限 5 条，防止记忆洪水。

**读取路径**：Profile 在会话开始全量注入，控制在 400 token 内；Preference 和 Episode 走工具按需检索，排序用 relevance × recency 衰减 × confidence，注入预算 800 token，超出截断。

**遗忘**：recency 做指数衰减，30 天未命中且置信度低的条目移入冷区定期清理。用户可以 list/del 任意记忆——可见可删是信任底线。

## 踩坑点

- **抽取太激进**：初期什么都存，一周后检索全是噪音。收紧到"明确、可执行的偏好才写"，噪音立减。
- **负向偏好丢失**：默认抽取只抓"喜欢什么"，"不要 xx"经常漏，prompt 里要显式要求抽取 prohibition 类条目。
- **相似度误召回**：聊"简洁架构"把"喜欢简洁回复"召回进来，加 tag 过滤和 type 约束比调 embedding 阈值有效。
- **并发写冲突**：两个会话同时更新同一 key，旧内容后到反而覆盖新值，改成单写者队列加版本号。

## 可复用建议

- 先定 schema 再谈向量，Profile 用固定 key 的 KV 能覆盖 80% 场景。
- 常驻层宁小勿大，按需检索优于全量注入。
- 每条记忆必带 `created_at / last_used_at / confidence`，没有时间戳的记忆系统没法修。
- 建一个 20 条的召回评测集（"我的 commit 规范是什么"之类），每次改动跑一遍防回归。
- 以 MCP tool 形式实现，任何客户端都能挂载，存储层留接口可替换。

## 总结

记忆系统的收益大头在写入端的结构化和过滤，不在检索端的花活。把偏好当一等数据管理——有 schema、有版本、有生命周期、对用户可见——助手才可能真正"记住你"。先从一张 400 token 的 Profile 卡开始，比一步到位的"无限记忆"务实得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/f1dcaf4341950b12.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/b07ed7c3bde18437.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/b928c86801a5220d.png)

