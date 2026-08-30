---
title: OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话
feedId: 35349
source: 综合讨论
publishedAt: 2026-08-30
---

## 背景

OpenClaw 引入子 Agent、MCP 工具或插件后，主会话很容易被“执行过程”污染。子 Agent 的中间推理、工具调用记录、MCP server 返回的大段 JSON、错误堆栈，常常一路回灌到主对话。主上下文窗口被快速占满，后续推理开始丢指令、跑偏或重复。

这通常不是模型能力问题，而是 session 边界没有守住：把“执行过程”当成“结果”返回了。

## 问题

典型表现包括：

- 主会话出现大量 child trace、tool_output、HTML 片段等非最终结果；
- 多轮后模型忘记最初任务，关键指令被挤到上下文外；
- 子 Agent 报错后，错误堆栈长期留在主会话，修复后仍影响后续生成；
- 嵌套子 Agent 让上下文接近指数级膨胀。

核心问题只有一个：主会话承担了太多它不需要知道的细节。

## 做法/步骤

### 1. 子 Agent 使用独立 session

不要让子 Agent 直接写入主 session。每个子任务创建独立 `session_id`，主会话只保存引用或简短摘要：

```json
{
  "task_id": "sub-20260117-01",
  "summary": "已检查 3 个文件，发现 2 个权限问题",
  "child_session_id": "sess-xxxx"
}
```

原始过程留在子 session，需要排查时再按 `child_session_id` 拉取。

### 2. 返回模式设为 final_only

子 Agent 的提示词或启动参数应明确：只返回最终结论，不返回思维链、不返回完整工具输出。工具输出先做摘要、截断或字段筛选，再回到主会话。

在 OpenClaw 里优先用 session 配置或子 Agent 启动参数实现，而不是只靠 prompt 硬约束。

### 3. 结构化返回结果

主会话不适合直接吃大段自然语言。子 Agent 返回一个精简对象：

```json
{
  "ok": true,
  "summary": "已完成检查，发现 2 个问题",
  "artifacts": ["report.md"],
  "error": null
}
```

主会话只需要 `summary`、`artifacts` 和 `error`，不需要过程。

### 4. 关闭不必要的继承

子 Agent 不要默认继承主会话全部 memory、工具状态和插件记录。只传入必要输入；需要共享的信息通过显式 payload 传递。

### 5. 设置上下文预算

给主 session 设 `max_context_tokens` 或清理策略。长工具输出按字节截断，MCP 返回只保留关键字段，避免整页 HTML 或完整日志进入主会话。

## 踩坑点

- **final_only 过度导致排障困难**：子 session 必须保留。主会话或外部日志里要记录 `child_session_id`，否则出错只能重跑。
- **memory 隐式共享**：部分插件或 MCP 会默认写全局 memory。子 Agent 结束后要清理状态，或使用独立 namespace。
- **错误堆栈回传**：报错时只回传 error code + 摘要，堆栈留在子 session。
- **嵌套深度失控**：多层子 Agent 会破坏隔离效果。设置 `max_depth`，每层都只返回摘要。
- **输入被误截断**：清理主上下文时，不要先把子 Agent 的输入截了。先保障子任务输入完整，再裁其他历史。

## 可复用建议

把子 Agent 当纯函数用：入参明确、出参结构化、内部日志留在局部、不外泄过程。主会话只存引用和结果摘要。

更推荐做一个 session hygiene 中间件或插件，自动完成三件事：

1. 删除超过阈值的 tool_output；
2. 把 child trace 移出主 session；
3. 只保留最近 N 轮关键消息。

这样比在每个 prompt 里反复提醒更稳定。

## 总结

子 Agent 污染主会话，不是靠更长上下文解决，而是靠显式边界：独立 session、final_only 返回、结构化摘要、关闭隐式共享。主会话保持轻量，执行细节留给子 session。上下文更耐用，排障链路也更清晰。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/6bf5ed67cd0bec94.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/6914fcaebd2c8661.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/7e63a8d60864ffdd.png)

