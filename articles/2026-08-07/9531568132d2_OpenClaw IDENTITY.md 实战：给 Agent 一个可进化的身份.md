---
title: OpenClaw IDENTITY.md 实战：给 Agent 一个可进化的身份基座
feedId: 31915
source: 综合讨论
publishedAt: 2026-08-07
---

## 为什么默认的 system prompt 不够用

在 OpenClaw 这类 Agent 框架中，通常用 system prompt 来约束行为、注入角色特征和工具调用规则。但随着任务复杂化，这种一次性注入的人格卡存在明显局限：

- **无法跨会话延续**：每次新对话都回到初始角色，Agent 的“经验”全部丢失。
- **团队协作时对齐成本高**：多个开发者共用同一 Agent 时，需要反复口头传达偏好，无法形成一致的工作风格。
- **动态适配能力弱**：面对新任务域时，固定 prompt 要么过于泛化，要么需要频繁手工调整。

我们真正需要的，是一个 **可读写、可演进、跨会话持久化的身份层**。这就是 IDENTITY.md 的核心设计意图。

## IDENTITY.md 是什么

IDENTITY.md 并不是一个花哨的概念，而是一个落地的文件约定：在项目或 Agent 工作目录下放置一个 Markdown 文件，其中结构化地描述 Agent 的**当前身份状态**——包括核心原则、领域知识摘要、行为偏好、已知陷阱、会话记忆摘要等。Agent 在每次推理前读取该文件，并可在触发条件满足时**自己更新该文件**，从而实现渐进式的自我对齐与知识积累。

它最初在 OpenClaw 社区的一些长期运行 Agent 实践中被提炼出来，现在已被部分接受为类似“runtime personality store”的轻量标准。

## 实践步骤：给 OpenClaw Agent 接入 IDENTITY.md

这里以 OpenClaw + MCP 工具链为例，给出最小可复现步骤。

### 1. 创建初始 IDENTITY.md

在 Agent 工作目录 `~/.openclaw/agents/assistant/` 下新建 `IDENTITY.md`，初始内容示例如下：

```markdown
# Agent Identity

## Core Principles
- Prefer explicit, minimal code examples
- When debugging, propose 3 hypotheses before asking for more logs
- Never fabricate config paths; always check with `read` tool first

## Domain Summaries
<!-- AGENT:UPDATE domain=kubernetes -->
K8s 常见问题模式：CrashLoopBackOff 80% 为启动命令错误或缺少配置卷。
<!-- /AGENT:UPDATE -->

## Behavior Adaptations
- None yet

## Known Pitfalls
<!-- AGENT:UPDATE type=pitfall -->
- 使用 `kubectl exec` 前需确认容器内是否有 shell
<!-- /AGENT:UPDATE -->

## Session Memory (last 3 interactions)
- 无
```

关键设计是使用 `<!-- AGENT:UPDATE ... -->` 标记块，让 Agent 在更新时有明确锚点，避免污染整个文件。

### 2. 在工具链中集成读写能力

通过 OpenClaw 的 plugin 机制或自定义 MCP 工具，暴露两个函数：

- `read_identity()`: 读取并解析 IDENTITY.md，分割出不同区块注入到当前 system prompt 底部。
- `update_identity(section, content)`: 接受区块名称与新内容，替换对应 `AGENT:UPDATE` 标记内的部分。

在 system prompt 中增加如下指令：

```
You have an evolving identity file at ~/.openclaw/agents/assistant/IDENTITY.md.
After a session where significant new domain insight, user preference, or mistake pattern is discovered, call update_identity to persist it.
Only update when confidence is high, and limit changes to the specific marked blocks.
```

### 3. 定义更新触发策略

无节制的更新会让 IDENTITY.md 膨胀并降低可用性。实践中落地了三类触发规则：

- **任务复盘钩子**：在长任务完成后（如排障线程关闭时），Agent 自动询问：“本次会话有无需要我记住的工作习惯或新知识？”得到确认后再写入。
- **错误纠正自动捕获**：当用户连续两次指出同类错误（如“又用错参数了”），Agent 触发一次更新，在 `Known Pitfalls` 区块追加条目。
- **阈值保护**：单次会话最多触发 3 次更新，每个区块内容长度不超过 500 字，超出则摘要化。

### 4. 版本控制与回滚

将 IDENTITY.md 纳入 Git 仓库（Agent 工作目录也建议用 Git 管理）。每次 Agent 写入时自动生成 commit，message 格式为 `identity: update <section> based on session <id>`。如发现 Agent“学坏了”，可以通过 `git revert` 快速回滚到上一个稳定身份状态。这也方便团队复现 Agent 当时的决策上下文。

## 踩坑记录

在实际持续跑了 2 周的 Agent 实例中，遇到以下典型问题：

- **循环自我强化**：Agent 在 `Domain Summaries` 里写下“大部分问题是 DNS 解析”，随后对新问题都优先怀疑 DNS，即便现象不相关。这是因为带有偏差的记忆被反复读取、放大。**解法**：在 system prompt 中强调“IDENTITY.md 是辅助参考，不是唯一决策依据”，并定期人工清理偏误总结。
- **并发写入冲突**：同一 Agent 被多个 MCP 客户端同时触发，出现文件覆盖导致丢失更新。**解法**：使用基于文件锁的写入（如 `flock`），或通过单一写入队列串行化更新请求。
- **隐私泄露风险**：测试时发现 Agent 把用户提供的内部主机名写入了 `Known Pitfalls`，导致文件推送到远程仓库后暴露。**解法**：在 `update_identity` 函数中增加敏感信息过滤器，禁止写入匹配 IP、hostname 模式的内容，并强制人工审核 `diff` 后再 push。

## 可复用建议

- **从最小化区块开始**：不要一下子定义十几种更新类型。从 `Known Pitfalls` 和 `Behavior Adaptations` 两类入手，验证价值后再扩展。
- **与 MCP memory 工具互补**：IDENTITY.md 适合存储慢变、高价值的结构化知识，而短期记忆（如当前任务上下文）仍应留在 MCP 的 `memory` 插件或会话向量库中。两者分层，避免把所有内容塞进一个文件。
- **加入元数据时间戳**：每个更新块附带 `last_updated: 2025-01-...`，Agent 在决策时可据此判断知识的新鲜度，自动忽略过期条目。
- **团队共享一个“基础身份”**：如果多个开发者使用同一 Agent，可以维护一个 `IDENTITY.base.md` 作为团队共识部分，允许 Agent 在此基础上叠加个人适配，减少冲突。

## 总结

IDENTITY.md 不是银弹，而是对“Agent 长期经验积累”问题的一种极简、可控的文件基座方案。它的优势在于利用了现有的文本接口和 Git 生态，不引入额外数据库或复杂服务；风险则主要来源于更新策略的设计不当和内容治理缺失。如果你正在维护一个需要连续工作数天甚至数周的 OpenClaw Agent，或是希望团队成员间的 AI 助手逐渐“懂你”，值得花半天时间尝试这一模式。让身份可进化，前提是进化路径被小心地工程化。

---

