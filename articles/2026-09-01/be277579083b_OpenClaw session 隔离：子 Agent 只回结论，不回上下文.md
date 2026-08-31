---
title: OpenClaw session 隔离：子 Agent 只回结论，不回上下文
feedId: 35626
source: 综合讨论
publishedAt: 2026-09-01
---

## 背景

在 OpenClaw 上做多 Agent 编排时，很容易把子 Agent 当成普通函数调用：在主 session 里直接塞任务，然后拿回一段长文本。结果主 Agent 的上下文里混进了子 Agent 的中间推理、工具原始输出、甚至内部系统提示词，导致主 Agent 决策变差、token 消耗飙升。这个问题在子 Agent 调用 MCP 工具或浏览器自动化时尤其突出，返回内容可能包含大量 DOM、日志和错误堆栈。

我们团队在跑一个价格监控 Agent 时踩过最典型的坑：让子 Agent 做网页信息抽取，它把整页 HTML 和重试日志都带回主会话，主模型直接开始分析 HTML 结构，而不是使用抽取后的摘要。最终跑一次任务的 token 成本翻了 4 倍，主 Agent 还经常被错误堆栈带偏。

## 问题拆解

子 Agent 的“污染”通常来自三个地方：

1. **共享 session 历史**：子 Agent 能看到主会话全部内容，也能往主会话写入消息。
2. **回传内容未裁剪**：把工具原始输出直接透传给主会话。
3. **复用 session_id**：并发子 Agent 用同一个 session_id 导致消息交叉写入。

## 做法/步骤

### 1. 给子 Agent 分配独立 session

创建子 Agent 时，显式指定新的 session_id，并关闭上下文继承。示例为伪代码，具体以你使用的 OpenClaw 版本为准：

```python
sub = openclaw.create_subagent(
    task="从商品页提取价格信息",
    session_id=uuid4().hex,       # 独立 session
    inherit_context=False,        # 不继承主会话
)
result = sub.run()
sub.close()
```

关键是 `inherit_context=False`（或等价配置），避免子 Agent 读取主会话历史。如果当前版本没有这个参数，可以手动创建一个空 context 对象传入。

### 2. 定义回传协议：只返回结构化字段

子 Agent 完成后，不要返回完整 message list，而是提取 result 中的指定字段。例如：

```json
{
  "status": "ok",
  "summary": "提取到 3 个价格数据，平均价 42.7",
  "artifacts": ["s3://bucket/result.json"],
  "error": null
}
```

主会话只接收这个 JSON，而不是子 Agent 的完整聊天记录。OpenClaw 的 subagent 结果对象通常包含 `final_output`，不要使用 `history` 作为回传内容。

### 3. 对工具输出做中间层裁剪

如果子 Agent 调用 MCP 工具，工具原始输出不要直接进主上下文。可以在子 Agent 内部加一个输出后处理：截断长文本、移除 HTML/日志、只保留关键字段。例如设置 `max_tool_result_chars=2000`，或通过自定义 wrapper 过滤掉非必要内容。

### 4. 控制子 Agent 的上下文预算

即使独立 session，子 Agent 自己也可能读取过长的工具输出而跑偏。给子 Agent 设置较小的 context window 或 `max_history_tokens`，强制它只使用必要信息。

### 5. 用摘要代替原文

如果必须回传一些上下文，让子 Agent 在结束时生成一段不超过 200 字的摘要，主会话只接收摘要。这比直接回传原始记录安全得多。

## 踩坑点

- **复用 session_id**：我们最早图省事用主 session_id 创建子 Agent，结果子 Agent 的所有中间消息都写进主会话，回滚都回不干净。
- **错误堆栈直接回传**：子 Agent 失败时，异常信息包含完整调用栈和内部提示词，主 Agent 看到后容易重复尝试或产生幻觉。应该捕获异常，只回传 `error_code` 和 `user_facing_message`。
- **忽略并发隔离**：多个子 Agent 并行跑时，如果共用一个 session 或共享内存，输出会交叉。务必给每个子 Agent 独立 session 和独立工作目录/临时文件。
- **过度隔离导致信息丢失**：完全不回传上下文，主会话无法判断子 Agent 是否可信，所以需要约定最小回传字段（状态、摘要、产出物引用）。

## 可复用建议

1. 封装一个 `run_isolated_subagent(task, max_tokens=1500, timeout=60)` 函数，默认独立 session、结构化回传、超时和 token 上限。
2. 主会话侧只读取 `result.summary` 和 `result.artifacts`，其余字段一律丢弃。
3. 在 CI 或测试里加一个“污染检查”：断言主会话上下文长度在子 Agent 运行后增长不超过预设阈值（例如 500 tokens）。
4. 给子 Agent 的 prompt 里明确写：“不要返回原始工具输出，只返回结论和必要引用”。

## 总结

OpenClaw 的 session 隔离不是把上下文完全切断，而是控制信息回流的方向和粒度。独立 session 负责“干活不串味”，结构化回传负责“带话不带噪声”。只要做到这两点，子 Agent 就能真正成为主 Agent 的可靠外挂，而不是上下文里的定时炸弹。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/d96749da90344fba.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/1bf23f8840635439.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/8591468232c255cd.png)

