---
title: OpenClaw 子 Agent 会话隔离：别让子任务把主上下文带偏
feedId: 33072
source: 综合讨论
publishedAt: 2026-08-14
---

## 背景

在 OpenClaw 里做多 Agent 编排时，主 Agent 通常负责拆解任务、分派子 Agent、汇总结果。子 Agent 可能去做代码检索、文件处理、MCP 工具调用、网页抓取等脏活累活。

但很多实践者会发现一个问题：子任务执行完后，主 Agent 的上下文变得非常“脏”。主会话里混进大量子 Agent 的工具日志、中间推理、MCP 返回片段，甚至重试记录。主 Agent 开始误判、重复调用工具，或者把子任务的局部结论当成全局事实。

这不是幻觉，而是会话隔离没做好。

## 问题

OpenClaw 里如果子 Agent 没有独立 session 边界，默认行为容易把子执行过程合并回主会话。常见表现：

- 主上下文快速膨胀，token 成本上升；
- 关键指令被大量中间输出淹没；
- 主 Agent 误读子 Agent 的工具调用结果；
- 子 Agent 内部的 MCP 连接状态、缓存、错误堆栈串进主流程；
- 多个子任务并发时，返回内容互相污染。

根因通常不是“子 Agent 不能用”，而是返回协议太宽：所有 messages、tool_logs、debug 信息都被带回主 session。

## 做法/步骤

我的做法是给每个子 Agent 建立明确的会话边界，并限制它只返回主 Agent 真正需要的东西。

### 1. 子 Agent 使用独立 session namespace

在 OpenClaw 的子 Agent 配置里，不要让子 Agent 直接写主 session。给子任务分配独立 namespace，例如：

```yaml
subagent:
  session:
    isolated: true
    namespace: "sub-{task_id}"
```

这样子 Agent 的上下文、工具调用记录、错误重试都留在自己的 session 里，主 session 只通过受控接口读取结果。

### 2. 定义最小返回协议

子 Agent 完成后，不要返回完整历史。只允许返回一个结构体：

```json
{
  "status": "success",
  "summary": "在 repo 中找到 3 处调用点，主要影响 auth 模块",
  "artifacts": ["artifact://sub-1234/result.json"],
  "errors": []
}
```

关键字段：

- `status`：成功/失败/部分完成；
- `summary`：压缩后的结论，限制在 300-500 token 内；
- `artifacts`：大结果存文件，只返回引用；
- `errors`：只保留可操作的错误信息，不返回完整堆栈。

### 3. 大结果走 artifact，不走上下文

子 Agent 生成的代码片段、JSON、表格、日志，不要直接拼进主会话。写入临时文件或对象存储，把引用返回给主 Agent。主 Agent 需要时再读，不需要就不占用上下文。

这比“截断到前 2000 字符”更安全，因为截断仍然可能带回格式错误的片段，且主 Agent 容易误读。

### 4. 隔离 MCP 工具作用域

如果子 Agent 复用主 Agent 的 MCP 连接，读写操作可能直接污染主流程。例如子 Agent 写文件、改数据库、更新 issue，主 Agent 事后并不知道状态已经变了。

我的配置是：

- 子 Agent 默认只读工具，写操作需要显式声明；
- 子 Agent 使用独立 MCP 连接或独立 credentials；
- 如果必须共享连接，返回协议里要带上 `side_effects` 字段，说明外部状态变化。

### 5. 设置执行预算和截断策略

给每个子 Agent 设置硬限制：

```yaml
limits:
  max_steps: 12
  token_budget: 8000
  truncate: "head+tail"
```

超过预算就终止执行，只返回 `status: "partial"` 和已完成的 artifacts，避免无限重试拖垮主会话。

## 踩坑点

实际落地时，这几个坑比较常见：

### 坑 1：只隔离 messages，没隔离工具

配置了 `isolated: true`，但子 Agent 仍然调用主 session 的 MCP 连接。结果上下文没串，工具状态串了，排查起来更隐蔽。

### 坑 2：返回 summary 本身变成大段文本

有人把子 Agent 的“摘要”写成了 3000 token 的复述。主 Agent 不但没省上下文，反而多了一层语义损耗。summary 应该是结论，不是精简版全文。

### 坑 3：错误重试导致子 session 历史被反复合并

子 Agent 失败后重试，如果重试逻辑把多次执行历史都返回给主 Agent，主 session 会看到矛盾的时间线。正确做法是只返回最后一次成功状态和最终 artifacts。

### 坑 4：并发子 Agent 共用 namespace

多个子任务共用 `namespace: "sub-default"`，日志交叉、artifact 覆盖、工具缓存互串。一定要用 task_id 或 trace_id 做命名空间隔离。

### 坑 5：base64 内联返回

文件不大时，容易图省事把 base64 编码后直接放回返回协议。主 session 迅速爆炸，而且不好调试。artifact-first 原则要执行到底。

## 可复用建议

如果你现在正在用 OpenClaw 做多 Agent 编排，建议把这套规则固化进 prompt 或配置模板：

1. **子 Agent 返回 schema 统一**：所有子任务遵守 `status/summary/artifacts/errors` 结构。
2. **artifact-first**：超过 500 token 的结果一律写文件，返回引用。
3. **工具作用域分离**：子 Agent 默认只读，写操作要显式审批或声明 side effects。
4. **命名空间唯一**：`task_id`、`trace_id` 必填。
5. **主 session 增长监控**：每轮执行后统计主 session token 增量，如果异常增长，优先排查子 Agent 返回路径。
6. **回归测试**：模拟子 Agent 执行后，断言主 session 不包含子工具日志、堆栈、重试记录。

## 总结

OpenClaw 的 session 隔离不是简单打开一个 flag 就能完成。它需要三个边界同时成立：会话上下文边界、返回协议边界、工具作用域边界。子 Agent 可以自由地跑复杂流程，但主 Agent 只拿到它需要的结论和引用。

把子 Agent 当成一个“有副作用的函数”，输入输出尽量通过 artifact 传递，而不是通过上下文堆叠。这样主会话干净了，token 成本下来了，排障路径也会清晰很多。

---

