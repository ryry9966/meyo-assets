---
title: OpenClaw 的 session 隔离：子 Agent 不污染主会话的工程化做法
feedId: 33862
source: 综合讨论
publishedAt: 2026-08-19
---

## 背景

在 OpenClaw 上做自动化，常见模式是主 Agent 做任务拆解，再通过子 Agent 或 MCP 工具去执行独立子流程。比如主 Agent 负责“分析仓库并生成修复计划”，子 Agent 负责“跑测试、收集报错、返回结果”。

问题也很典型：子 Agent 在跑批量任务时，会把每一步工具调用、stdout、中间推理、重试日志都带回主会话。主 Agent 拿到这些过程信息后，很容易把中间态误判成最终结果，或者上下文被长日志塞满后忘记原始目标。

这不是“子 Agent 输出太多”的问题，而是主会话同时承担了控制面和执行面，导致 process 级上下文与 result 级上下文混在一起。

## 问题表现

- token 消耗陡增，主 Agent 几轮后开始丢上下文
- 子任务中间日志被主 Agent 当成结论继续推理
- MCP 工具返回 500 行 stdout，主 Agent 无法定位关键错误
- 长任务执行时，主会话被阻塞，后续交互质量下降

核心就一句：**子 Agent 的过程数据不能直接进主会话，主会话只需要结果摘要与引用。**

## 做法/步骤

### 1. 先定返回协议，不要回传全量 transcript

在 OpenClaw 里不要让子 Agent 把完整执行历史 return 给主会话。优先把子 Agent 的返回模式设置成 final only，只保留最终输出。如果没有现成开关，就在子 Agent 的最后一步强制走结构化返回：

```json
{
  "status": "ok",
  "summary": "3 tests failed, root cause: fixture path mismatch",
  "error_code": null,
  "payload_ref": "artifact://subagent/run-20250611-01/result.json",
  "duration_ms": 18230
}
```

主会话只读这个信封，不读原始过程。

### 2. 会话标识隔离

给每个子 Agent 分配独立 `session_id`，通过 `parent_session_id` 关联主会话。主会话里只记录子会话句柄，不记录内容。

工程上要避免一个常见问题：子 Agent 为了省事直接复用主会话的 default session，这样隔离就失效了。OpenClaw 里只要 session store 允许自定义 id，就显式传入，别用全局单例。

### 3. MCP 工具输出做截断和落盘

很多污染来自 MCP server 返回的 stdout/stderr，而不是子 Agent 自己的推理。建议在 MCP wrapper 层做输出收敛：

- stdout 只回传尾部 20 行
- stderr 只回传尾部 10 行
- 完整输出写入对象存储或本地 artifact 目录
- 返回体里只带 `artifact_uri` 和 `exit_code`

如果 MCP server 是自研的，可以在 server 内部完成；如果是第三方插件，就在 OpenClaw 与 MCP 之间加一层转发。

### 4. 主 Agent 只消费摘要

主 Agent 拿到的结果，只保留：

- `status` / `ok`
- `summary`
- `payload_ref`
- `error_code`
- `last_stderr`

不要让主 Agent 在后续推理里直接阅读完整日志。需要排查时，通过 `payload_ref` 或子 session 去单独取，不要在对话链路里展开。

### 5. 长任务异步化

超过一定时长的子任务，不要让主会话同步等待。可以改成：

- 子 Agent 独立执行
- 完成结果写入 artifact
- 回调或通知机制告知主 Agent
- 主会话只收到一个完成通知和引用

这样主会话不会被长执行过程占用，隔离效果更稳定。

## 踩坑点

### 完全隔离后排查变成黑盒

如果子 Agent 什么过程都不留，出错后你会很难定位。正确做法是：**完整日志留在子 session 和 artifact 里，主会话只拿摘要。** 调试时手动拉取子 session，而不是让主 Agent 自己去读全量。

### 只返回 `ok: false` 没有用

很多封装为了简洁，失败时只回一个失败状态。结果主 Agent 不知道原因，只能反复重试。失败信封里至少要带 `error_code` 和 `last_stderr`，否则后续推理没有抓手。

### 子 Agent 继承了错误的人设

子 Agent 如果继承了主 Agent 的 system prompt，可能会出现“我也是一个主协调者”的认知错位。它会重复规划，而不是执行。给子 Agent 单独写 system prompt，并关闭不必要的主会话工具权限继承。

### 并发写同一个 parent session

如果多个子 Agent 同时把结果写回同一个父会话，可能出现竞态。建议 parent 只读，子 session 独立写；主会话合并结果时用 `session_id + timestamp` 做幂等。

### 用自然语言总结日志容易丢关键信息

不要把一大段原始日志压缩成“好像有问题”返回给主会话。结构化 JSON 更稳定，也更容易被后续工具消费。

## 可复用建议

- 建立统一的子 Agent 返回信封：`status`、`ok`、`payload_ref`、`summary`、`error_code`、`duration_ms`
- 给主会话预留子任务返回的 token 预算，比如不超过 800 token，超出部分强制落盘
- 所有子 Agent 调用走一个统一的 finalizer，不允许直接 return
- 日志分级：debug 进 session，info 进 artifact，error 回主会话
- 主会话只当控制面，子会话当执行面，不要在语义上混淆

## 总结

子 Agent 污染主会话，不是靠“让子 Agent 少说点”能解决的。关键是把主会话和子会话的职责分开：主会话做决策和编排，子会话做执行和过程记录。控制面只拿结构化摘要与引用，执行面保留完整过程。

这样处理后，主 Agent 在长链路任务里更不容易失焦，token 消耗会更可控，排查路径也清晰得多。

---

