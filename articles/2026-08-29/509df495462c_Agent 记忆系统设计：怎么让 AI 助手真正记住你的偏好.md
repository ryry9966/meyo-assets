---
title: Agent 记忆系统设计：怎么让 AI 助手真正记住你的偏好
feedId: 35146
source: 综合讨论
publishedAt: 2026-08-29
---

## 背景：Agent 为什么“记不住”

OpenClaw 或基于 MCP 的 Agent 默认是无状态的。每次任务启动，系统只带当前指令和工具描述。你对它说过“不要用 emoji”“代码优先给可运行示例”，下次它照样问或做错。问题不在于模型能力，而在于这些偏好没有被结构化保存、检索和注入。

## 问题拆解

要做一个能用的记忆系统，至少要回答四件事：

1. 记什么：稳定偏好、项目约定、术语映射，而不是临时任务状态。
2. 存哪里：JSON、SQLite、或 MCP resource。
3. 何时读/写：任务开始时读取，用户纠错或明确偏好时写入。
4. 怎么防止错误记忆：需要来源、置信度、作用域。

## 做法：一个轻量 memory 层

我建议从最小实现开始：一个 `memory.json`，通过 MCP server 暴露读写工具。结构如下：

```json
{
  "preferences": [
    {
      "key": "code_style",
      "value": "prefer explicit types, avoid clever one-liners",
      "source": "user_feedback",
      "confidence": 0.9,
      "scope": ["coding"],
      "updated_at": "2025-01-01T10:00:00Z"
    }
  ]
}
```

MCP server 提供两个工具：

- `save_preference(key, value, scope, confidence)`：写入或更新。
- `search_preferences(scope, query)`：按作用域检索。

在 OpenClaw 中注册：

```json
{
  "mcpServers": {
    "memory": {
      "command": "node",
      "args": ["memory-server.js"]
    }
  }
}
```

使用流程：

1. 任务启动，Agent 根据任务类型调 `search_preferences("coding", current_task)`。
2. 把命中的偏好注入 system prompt 或上下文片段。
3. 用户说“以后都这样”，Agent 调 `save_preference`。
4. 每晚或每 N 次会话，跑一个整理任务，合并重复 key，降低长期未命中的置信度。

## 踩坑点

- **临时决定被当成长期偏好**：某次任务选了 Python，不代表永远用 Python。写入时要分 scope，不要把任务状态写进 preference。
- **上下文膨胀**：偏好一多，注入全部会挤占推理空间。每次只注入与当前任务最相关的 5-8 条。
- **记忆污染**：Agent 自己推理出的“用户可能喜欢”不能直接写，至少标记 source 为 inferred，并给低置信度。
- **过期偏好**：用户习惯变了，旧偏好还在干扰。建议给偏好加 TTL 或定期让用户确认。
- **冲突处理没有兜底**：同 key 不同值，不要静默覆盖。保留旧值，提升新值的置信度，或标记为待确认。

## 可复用建议

- 结构统一：`key / value / source / confidence / scope / updated_at` 六个字段足够覆盖大多数情况。
- 命名空间隔离：按项目、团队或角色分 namespace，避免全局污染。
- 写入有门槛：只有用户明确要求、或同一模式重复出现三次以上才写入。
- 保存原始反馈：不要只存摘要，保留用户原话便于回溯。
- 优先走 MCP：把 memory 做成 MCP server 或 plugin，OpenClaw、其他 Agent、CI 流程都能复用。

## 总结

Agent 记忆系统的关键不是“存更多聊天记录”，而是把稳定偏好结构化成可检索、可更新、可隔离的小块，在正确时机注入。从 JSON + MCP 开始，加上置信度和 scope 控制，就能明显减少重复提问和错误建议。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/12fc22453d3befca.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/243d24ead137602f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/87d78d7afc9c1112.png)

