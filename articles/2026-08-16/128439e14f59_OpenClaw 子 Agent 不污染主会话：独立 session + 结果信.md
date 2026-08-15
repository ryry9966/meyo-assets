---
title: OpenClaw 子 Agent 不污染主会话：独立 session + 结果信封的隔离方案
feedId: 33316
source: 综合讨论
publishedAt: 2026-08-16
---

## 背景

在 OpenClaw 里用子 Agent 处理长任务、多步 MCP 调用或检索时，很容易出现一个问题：子 Agent 的中间推理、工具原始输出、系统提示、错误堆栈被直接 append 回主会话。主会话上下文快速膨胀，模型注意力被无关细节带偏，token 成本上升，甚至 MCP 返回的不可信内容会进入主控制流。

这不是“子 Agent 能力不够”，而是信息回流方式不对。工程上需要把子 Agent 的执行过程与主会话解耦，只有结构化结果能回到主会话。

## 问题表现

典型反模式：

1. 把子 Agent 完整 transcript 原样写回主 session；
2. 把主 session 全部历史复制给子 Agent 当上下文；
3. 子 Agent 和主会话共用同一个 memory namespace；
4. MCP 工具 raw output 不经过滤直接进入主上下文。

结果就是：主会话越来越长，关键指令被稀释；子 Agent 看到的敏感上下文过多，增加泄露风险；一次工具调用失败会带出一堆堆栈，影响后续规划。

## 做法 / 步骤

### 1. 给子 Agent 分配独立 session

不要在主 session 内直接“续写”子任务。为每个子任务创建独立 session，命名规范建议：

```text
sub:{parent_session_id}:{task_id}
```

其中 `task_id` 使用短随机 ID，避免并发冲突。子 Agent 的 system prompt 只包含完成当前任务所需的最小信息：任务说明、输入参数、可用工具白名单、返回 schema 约束。

主会话只记录一次 spawn 事件和最终结果引用，不记录中间过程。

### 2. 用结果信封回流

子 Agent 完成后，不返回完整对话记录，只返回一个结构化的结果信封：

```json
{
  "status": "ok",
  "summary": "完成数据清洗，共处理 1200 条记录",
  "artifacts": ["workspace://jobs/clean-8f3a/report.md"],
  "error": null,
  "metrics": { "tokens": 3120, "tool_calls": 7 }
}
```

在主会话里只追加这个 JSON，或者追加 `summary` + `artifacts` 路径。不要让子 Agent 的中间日志进入主上下文。

### 3. MCP 工具调用放在子 session 内

尤其当 MCP 工具返回大段网页、API 响应或文件内容时，不要让这些 raw output 直接回到主会话。让子 Agent 在子 session 里调用 MCP 工具，读完原始输出后生成摘要或写 artifact 文件，只把摘要和文件路径返回主会话。

### 4. 子 session 完成即归档

子任务结束后，将子 session 标记为 `archived`，保留元数据便于审计，但不再参与主会话上下文组装。避免 session 列表和上下文缓存无限增长。

## 踩坑点

**子 session 上下文过短导致结果跑偏。**  
独立 session 不代表可以只给一句“你看着办”。必须显式传入任务 brief、相关文件路径、输出 schema 和约束。否则子 Agent 会因为缺少上下文而自行补全，返回不可用结果。

**解析失败：子 Agent 输出 JSON 外带说明文字。**  
常见返回是：

```text
好的，结果如下：
{"status":"ok", ...}
```

这会破坏 schema 校验。可以在 final prompt 里要求“只输出一行 JSON，不要输出任何解释”，并在解析端做容错：提取第一个合法 JSON 块，失败则重试一次。

**主 session 历史被全量复制。**  
子 Agent 很少需要主会话的全部历史。只传裁剪后的 relevant context，避免敏感信息泄露，也降低 token 消耗。

**memory 共享污染。**  
如果 OpenClaw 启用了 memory，需要为子 session 设置独立 namespace，或直接关闭子 session 的 memory 写入。否则子 Agent 的记忆会混入主会话后续检索。

**超时和取消不彻底。**  
主会话等待子 Agent 返回时必须设置 timeout。超时后要显式取消子 session，否则子 session 可能继续执行并写日志，造成资源浪费和干扰。

## 可复用建议

封装一个 `runSubAgent(task, ctx)` 工具或函数，内部统一处理：

- session ID 生成与隔离；
- 最小上下文注入；
- MCP raw output 不回传；
- 结果信封解析与 schema 验证；
- 超时、取消、归档策略。

这样业务侧不需要每次手动管理 session 生命周期。

同时建议：

- 定义统一的 `SubAgentResult` schema，所有子 Agent 都遵守；
- 主会话只记录 `subResult` 引用或短摘要，不记录完整输出；
- 对大文件产物写 workspace / 对象存储路径，不要塞进上下文；
- 监控子 session 的 token 消耗、tool_calls 数量和错误率；
- 对不可信 MCP 输出保持边界：子 Agent 可以读取和总结，但不应直接改变主会话控制流。

## 总结

OpenClaw 的 session 隔离不是简单“另开一个对话”，而是把子 Agent 的执行上下文、工具输出、记忆与主会话彻底解耦。主会话只保留任务级信息和最终结果引用，过程细节留在子 session 内。

这样做的收益很明显：主会话更稳定、上下文更干净、成本更可控，也更容易审计子任务。对于需要频繁调用 MCP、插件或自动化流程的场景，这套模式可以作为默认工程约定。

---

