---
title: Agent 记忆系统设计：怎么让 AI 助手真正记住你的偏好
feedId: 36002
source: 综合讨论
publishedAt: 2026-09-04
---

## 背景

LLM 本身无状态，Agent 的“记性”完全靠外围系统。常见做法有两种极端：要么把历史对话全塞进 system prompt，token 成本线性上涨、关键指令被稀释；要么什么都不存，每次重开归零——你昨天才说过“用 pnpm”，今天它又开始 `npm install`。

我们在 OpenClaw 插件侧做了一个偏好记忆系统，跑了几个月，把收敛后的方案和踩过的坑整理如下。

## 问题：记什么、怎么取、谁来写

**1. 存储要分层**，不是所有记忆都一样：

- 偏好层：结构化 KV，如“包管理器=pnpm”“回复用中文”，量小（10~30 条），每次会话全量注入；
- 情节层：历史任务摘要，量大，embedding 检索，按需注入 Top-K；
- 工作层：当前会话上下文，不落盘。

分层的核心目的是控制注入预算：偏好层全量，情节层按相似度阈值 + 硬性 token 上限。

**2. 写入要有触发标准**：

- 用户显式说“记住……”→ 直接写，标记 confirmed；
- 用户纠正 Agent（“不是 npm，是 pnpm”）→ 触发 update 而非 append；
- Agent 自主提取的候选记忆 → 带置信度，低置信度先挂起，不自动生效。

**3. 工具形态**：用 MCP 暴露 `memory_write` / `memory_search` / `memory_delete` 三个工具，让 Agent 自主调用，但写入走统一管道：去重（embedding 相似度 > 0.92 拒绝插入、改为合并）、打时间戳、记录来源。

## 关键实现

- 存储：一张 SQLite 表足够，字段含 namespace、key、value、embedding、confidence、updated_at；向量检索用 sqlite-vec 即可，别急着上独立向量库。
- 命名空间：user / project / session 三级。全局偏好（“喜欢简洁回复”）和项目偏好（“这个仓库用 tab 缩进”）必须分开，否则跨项目互相污染。
- 读取打分：relevance × recency × frequency，偏好层除外。
- 遗忘机制：长期未命中且低置信度的记忆降级，定期 compaction 时由 LLM 批量判定合并/删除/保留。

## 踩坑点

- **记忆污染**：早期 Agent 会把“用户今天问了 Kubernetes”当偏好存下来，两周后库里七成是垃圾。解法：写入前过一道“这是偏好、事实，还是一次性上下文？”的分类判断。
- **重复与冲突**：同一偏好换措辞存了五遍；后来“用 pnpm”和“用 npm”并存，Agent 随机选。解法：插入前去重 + update 时显式 supersede 旧值，保留变更历史。
- **错误归因**：Agent 把用户帮同事调试 Python 推断成“主用 Python”。解法：推断类记忆需 ≥3 次独立命中才升级为生效。
- **隐私边界**：默认不存密钥、住址类信息；提供查看和删除入口，用户能看到 Agent 到底记了什么——这点比召回率更重要。

## 可复用建议

1. 先用 JSON/SQLite 手工维护 10 条偏好，能解决 80% 的“记不住”抱怨，再考虑自动化。
2. 把写入标准写成显式 prompt 规则，不要指望模型自己判断什么值得记。
3. 建一个 20 条左右的回归测试集（“我的包管理器是什么？”），每次改动跑一遍。记忆系统没有测试就是薛定谔的。
4. 写操作全量落日志，排查“它为什么这么懂/这么不懂”时非常有用。

## 总结

Agent 记忆本质是数据工程问题，不是 prompt 技巧：分层存储控制成本，显式写入标准控制噪声，supersede + compaction 对抗过期，命名空间隔离作用域。先小后大、可查可删、有测试，比一上来堆向量库靠谱得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/16ef34c62612017b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/0f5aef1a631c82b0.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/42d5b9fcd94a7474.png)

