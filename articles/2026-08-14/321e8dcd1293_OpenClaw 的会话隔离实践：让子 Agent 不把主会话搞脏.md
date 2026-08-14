---
title: OpenClaw 的会话隔离实践：让子 Agent 不把主会话搞脏
feedId: 33101
source: 综合讨论
publishedAt: 2026-08-14
---

# OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话

## 背景

在 OpenClaw 里接 MCP、做自动化或插件开发时，经常会把耗时或高噪声任务拆给子 Agent：批量抓取、代码生成、日志分析、外部 API 调试等。理想情况下，子 Agent 干完活回传一个干净结果，主会话继续决策。但实际跑起来，主会话经常被中间过程污染——子 Agent 的工具输出、重试日志、网页正文、原始 JSON 全混进上下文，主 Agent 注意力被带偏，token 消耗暴涨，后续动作也开始“漂”。

这不止是上下文长的问题。更隐蔽的是 session 状态、memory、工具结果被隐式共享，导致子 Agent 的一次失败写入，影响主会话后续所有判断。

## 问题定位

子 Agent 污染主会话，通常来自四个地方：

1. **上下文继承过度**：创建子 Agent 时，把主会话完整 history 传进去，子 Agent 带着旧结论跑，又把它自己的中间推理回传回来。
2. **工具结果直接回灌**：MCP 工具返回 30KB JSON，子 Agent 不处理就返回，主会话被迫消化。
3. **输出没有契约**：子 Agent 返回自然语言，而不是结构化摘要，主会话不知道该取哪些字段。
4. **状态隐式共享**：子 Agent 使用同一个 memory namespace，或者默认写入全局状态，污染主 Agent 的长期记忆。

## 做法

### 1. 给子 Agent 独立 session 命名空间

在 OpenClaw 里创建子任务时，不要复用主会话的 `session_id`。使用带任务类型的命名规则：

```text
sub:fetch_price:20250411_01
sub:codegen:uuid-4f2a
```

这样排查时能区分主会话和子会话，也能在任务结束后精准清理。

### 2. 裁剪上下文，而不是全量透传

子 Agent 只需要知道任务目标和最小输入，不需要主会话历史。伪代码思路：

```python
sub = ctx.spawn_session(
    namespace="sub:fetch_docs",
    history_policy="none",
    context={
        "task": "读取以下 3 个 URL，提取版本号和发布时间",
        "input": urls,
    },
    read_only_memory=True,
)
```

严格限制传入内容，避免把主 Agent 的前几轮分析带进去。

### 3. 强制返回契约

让子 Agent 按固定结构返回，而不是自由发挥。例如：

```json
{
  "status": "ok",
  "summary": "3 个页面解析完成",
  "data": [
    {"url": "...", "version": "2.1.0", "published_at": "2025-04-10"}
  ],
  "errors": [],
  "artifacts": ["runs/sub_fetch_docs_01/raw/1.html"]
}
```

在子 Agent 的 system prompt 里写死 schema，并限制返回字段数量。超过契约的内容不进入主会话。

### 4. 大输出落盘，不进入上下文

如果子 Agent 调用 MCP 工具拿到大 JSON 或长文本，不要在消息里直接返回。先写文件，再只回传路径和摘要：

```text
raw 结果已保存到 runs/sub_fetch_docs_01/raw/1.json
共 14200 字节，提取到 4 个关键字段。
```

主会话如果后续需要，再通过工具按需读取。不要让主会话为不需要的细节付 token。

### 5. 状态写回只走 return value

子 Agent 不要直接写主 memory 或全局状态。它的结论应该通过返回值交给主会话，由主 Agent 决定哪些信息值得写入长期记忆。这样即使子 Agent 跑飞，也不会破坏主会话的状态。

## 踩坑点

1. **namespace 隔离了，但工作目录没隔离**  
   多个子 Agent 并发写同一个临时文件，或者子 Agent 清理时误删主会话文件。建议按 `session_id` 分目录：`runs/{session_id}/artifacts/`。

2. **MCP 工具结果自动合并到父历史**  
   有些 MCP 连接器或框架默认把工具 output 写回父消息。需要在工具包装层做截断：  
   ```text
   max_output_chars = 4000
   ```
   超过部分落盘，只回传前 4000 字符 + 文件路径。

3. **子 Agent 的 stdout/日志被捕获**  
   子 Agent 内部打印的调试日志会变成返回内容。把运行日志写进独立文件，不要在 stdout 吐大段信息。

4. **并发子 Agent 共用同一个 session_id**  
   这会让消息串味、状态互相覆盖。务必保证每个子任务唯一 ID。

5. **返回契约失败时，raw 直接进了主会话**  
   要加一层校验：如果子 Agent 返回不符合 schema，只把 `status: "invalid_output"` 和截断后的前 500 字放入主会话，不把完整 raw 结果暴露。

## 可复用建议

做一个固定模板，避免每次手写隔离逻辑。推荐封装一个 `run_isolated_subagent` helper，统一处理：

- 独立 `session_id` 和 namespace
- `history_policy="none"`
- 上下文只传任务和输入
- 输出 schema 校验
- 工具输出截断与落盘
- 子 Agent 不写主 memory
- 任务结束主动关闭 session

配置层建议：

```yaml
subagent:
  history: none
  memory: read_only
  output:
    require_schema: true
    max_tokens: 1500
  artifacts:
    path: "runs/{session_id}/"
  tools:
    allowlist: true
    max_output_chars: 4000
```

## 总结

子 Agent 隔离不是“能不能跑”的问题，而是“跑完后主会话还能不能用”的问题。核心原则只有三条：**少看历史、少带回中间产物、状态只通过返回值交接**。在 OpenClaw 里做自动化时，先把这个边界建好，再考虑并行和复杂度，能省掉大量排障时间。

子 Agent 可以跑得脏，但主会话必须保持干净。

---

