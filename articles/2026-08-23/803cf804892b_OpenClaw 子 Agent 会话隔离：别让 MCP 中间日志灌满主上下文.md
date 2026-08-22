---
title: OpenClaw 子 Agent 会话隔离：别让 MCP 中间日志灌满主上下文
feedId: 34244
source: 综合讨论
publishedAt: 2026-08-23
---

## 背景

在 OpenClaw 里做复杂自动化时，主会话通常负责拆解任务、调度子 Agent、汇总结果。子 Agent 则去执行具体动作：查 MCP、跑脚本、读文件、调插件。问题在于，子 Agent 执行过程中的中间输出很容易顺着工具返回通道流回主会话。

常见表现是：主上下文越来越大，真正有用的决策信息被稀释；模型开始复述子 Agent 的调试日志；一次批量任务下来，token 成本明显上升，但主会话并没有更聪明。

## 问题

核心问题不是“子 Agent 不能有输出”，而是**没有做会话粒度和信息粒度的隔离**。

如果子 Agent 的 session 直接挂在主会话下，或者工具输出策略是 passthrough 模式，下面这些内容都会进入主上下文：

- MCP 查询返回的原始 JSON
- 子 Agent 重试时的中间报错堆栈
- 插件 debug 打印
- 文件读取的全文，而不是摘要
- 子 Agent 的内部推理过程

主模型看到这些后，容易把子任务的执行噪声当成当前任务的一部分，进而出现误判。更麻烦的是，一旦某个子任务失败，主模型可能试图“修复子 Agent 内部逻辑”，而不是继续推进主任务。

## 做法/步骤

### 1. 给子 Agent 独立 session 命名空间

不要让子 Agent 复用主会话的 sessionId。可以在 OpenClaw 中为每个子任务分配独立命名空间：

```yaml
subagent:
  session_id: "sub:{task_id}:{role}"
  parent: "main:{task_id}"
  inherit_context: false
```

这样做的目的是让子 Agent 拥有自己的上下文生命周期。主会话只知道“我派了一个任务”，不知道任务内部发生了什么。

### 2. 统一子 Agent 返回结构

子 Agent 完成或失败后，不要直接把内部日志返回。建议在子 Agent 侧做一层 finalize 封装：

```json
{
  "ok": true,
  "result": "已生成 3 份周报草稿，未发现重复文件。",
  "error": null,
  "metrics": {
    "tool_calls": 5,
    "tokens_used": 1800
  }
}
```

主会话只接收这个结构化结果。`metrics` 可以用于后续成本统计，但不参与主任务推理。

### 3. 主会话侧限制注入内容

在 OpenClaw 的子 Agent 调用工具中，配置输出映射规则。只允许 `result` 和 `error` 字段进入主上下文，屏蔽 `raw_log`、`debug_info`、`stack_trace` 等字段。

```yaml
tools:
  subagent_call:
    output_mode: "mapped"
    mapped_fields:
      - result
      - error
    max_output_tokens: 800
```

`max_output_tokens` 很重要。即使子 Agent 返回了结构，也要防止它把一篇长文塞进来。通常子任务回传控制在 300–500 token 足够。

### 4. MCP 工具输出降噪

很多污染不是来自子 Agent 本身，而是来自 MCP server 的原始响应。建议把 MCP 工具调用也放进子 Agent session 内执行，不让 MCP 输出直接回主会话。

具体做法是：主会话只调用一个 `subagent.run`，该子 Agent 内部再调用 MCP。子 Agent 负责消化 MCP 返回内容，输出人类可读摘要。主会话接触不到 MCP 原始 JSON。

## 踩坑点

### 1. 父引用没清干净

即使子 Agent 用了独立 session，OpenClaw 可能仍会记录 parent 引用。如果任务结束后不清理，查询上下文时还是可能把子会话内容带出来。建议在子任务完成后显式关闭 session，而不是等 TTL 自动过期。

### 2. 工具策略绕过隔离

有些插件或自动化脚本会直接向主会话写日志。尤其是 `console.log` 或 debug 输出，可能绕过子 Agent 的返回封装。需要检查插件侧的日志级别，或者在 MCP 客户端层统一设置 `output_filter`。

### 3. 错误回传过度

子 Agent 失败时，完整堆栈可能比成功结果还长。一旦这些堆栈进入主上下文，主模型容易陷入“调试子 Agent”的循环。建议错误回传只保留错误类型和一句话原因，详细日志落到文件或独立 trace session。

### 4. 等待超时日志累积

主会话等待子 Agent 时，如果每次轮询都往上下文里加“仍在等待”，token 会白白损耗。等待逻辑应放在工具层，主上下文只记录最终状态。

## 可复用建议

- 建立统一的子 Agent 返回 Schema，所有子任务强制走 `{ok, result, error, metrics}`。
- 在子 Agent 提示词里写明：“只返回结论和必要证据，不得返回原始工具输出。”
- 为主会话设置提示词约束：不允许展开子 Agent 内部日志，不允许推测子 Agent 失败原因。
- 定期清理 orphan session，尤其是失败任务的残留 session。
- 监控主会话的上下文增长速度。如果一次批量任务后主上下文增长异常，优先检查 MCP 和子 Agent 输出策略。

## 总结

Session 隔离不是把子 Agent 变成黑盒，而是控制信息回流到主会话的粒度和生命周期。主会话需要的是决策信息，不是执行过程。把原始输出留在子 session，把结论和必要的错误信号带回主 session，才能让 OpenClaw 的自动化任务保持稳定、可控、可维护。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/3b681f1d13bbae38.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/7e3152c5f80852b9.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/82b7355dda5201eb.png)

