---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 34484
source: 综合讨论
publishedAt: 2026-08-24
---

## 背景

在 OpenClaw 上接 MCP、插件和自动化任务后，最常遇到的不是模型能力问题，而是“失忆”。每次新会话都要重新交代偏好、边界和工具权限，Agent 表现时好时坏。后来我发现，缺的不是更长的 system prompt，而是一个持久化的身份文件。

## 问题

我维护一个 OpenClaw 自动化项目，定时执行数据检查、调用 MCP 工具生成报告。主要问题有三个：

1. **身份漂移**：新会话有时过于保守，有时又敢直接调用删除类工具。
2. **上下文浪费**：大量偏好堆在系统提示里，任务还没开始就消耗几千 token。
3. **维护困难**：插件授权、MCP 工具边界散落在不同脚本和 prompt 中，改一处要全局找。

核心矛盾是：Agent 需要稳定身份，但静态 prompt 不适合承载会变化的记忆和边界。

## 做法

我把 IDENTITY.md 当作 OpenClaw 的身份层，分四步落地。

### 1. 创建并加载

在项目根目录创建 IDENTITY.md。如果 OpenClaw 已支持，会话启动时会自动注入；如果不支持，可以在系统提示顶部加一句：

```text
Read IDENTITY.md as your persistent identity. Follow its constraints before any task.
```

这样每次会话都先读取身份文件，而不是把身份写死在系统提示里。

### 2. 结构化字段

避免写成散文，只保留几块核心字段：

```markdown
# Identity
- Role: OpenClaw automation operator
- Project: data-pipeline

# Decision_Boundaries
- Never delete files without explicit approval.
- MCP tools allowed: read, search, report.
- MCP tools denied: write, execute.

# Preferences
- Use concise Chinese reports.
- Prefer JSON output for logs.

# Evolution_Log
- 2024-11-02: stopped auto-retrying after 3 failures.
- 2024-11-05: added data retention boundary.
```

Identity 是静态身份；Decision_Boundaries 是硬约束；Evolution_Log 是演进记录。Preferences 可按需保留。

### 3. 受控更新

“可进化”不等于“随便改”。我在文件末尾加约束：

```text
If you discover a repeated failure or missing boundary, propose an update via git diff. Do not modify this file directly during task execution.
```

Agent 只输出 diff，人工 review 后合并。这样身份可以进化，但不会被单次幻觉污染。

### 4. 版本控制

IDENTITY.md 必须进 git，每次变更单独提交，message 用 `identity: add retry boundary` 这种格式。行为异常时可以快速回滚，并把 Evolution_Log 和 git log 对照，定位是哪次身份变更引入了问题。

## 踩坑点

### 文件膨胀

初版写了 800 多行，光身份文件每次会话吃掉 2000+ token。后来限制在 200 行内，只保留当前边界和最近 5 条演进记录。

### 自我修改循环

无限制更新会让 Agent 把偶发错误写成永久规则，比如一次网络超时后写“Never use network”。后来改为：连续出现 3 次且无法用重试解释的问题，才允许提交身份变更。

### 格式破坏

Agent 可能把字段改名或改写成散文。我在顶部加格式约束：

```text
Format: Markdown with fixed top-level fields. Do not rename fields.
```

CI 里再加简单脚本检查必需字段是否存在，防止格式被悄悄破坏。

### 敏感信息

不要放 API key、内网地址、账号。IDENTITY.md 每次会话都会被注入，泄露面比普通文件更大。敏感信息放环境变量或 MCP 配置里。

## 可复用建议

- 控制在 200 行内，字段固定，只放当前有效的身份和边界。
- 硬约束放 Decision_Boundaries，软偏好放 Preferences，不要混在一起。
- 身份变更必须 diff + review，禁止 Agent 直接覆盖原文件。
- 配合 MCP 工具权限，身份文件里的 deny 列表要和实际权限一致。只写 deny 但 MCP 仍开放 write，等于没有边界。
- 每 1–2 周清理一次 Evolution_Log，过时规则归档到 CHANGELOG.md，保持文件精简。

## 总结

IDENTITY.md 解决的不是“让 Agent 更聪明”，而是“让 Agent 更稳定”。它把身份、边界和演进记录从系统提示中剥离，形成一个可维护、可回滚、可审计的长期层。对 OpenClaw 项目来说，先让 Agent 知道它是谁，再让它做事，往往比堆插件和 prompt 技巧更有效。

如果你的 OpenClaw 项目还在反复调教，可以从最小版本开始：Role、Deny、Evolution_Log，三个字段就够用。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/fd335b73d4f4cce9.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/c1d36c586596679d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/bc5f748c9cfafb98.png)

