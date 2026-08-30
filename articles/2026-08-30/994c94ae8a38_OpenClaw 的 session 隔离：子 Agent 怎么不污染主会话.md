---
title: OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话
feedId: 35421
source: 综合讨论
publishedAt: 2026-08-30
---

# OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话

## 背景

在 OpenClaw 里接子 Agent 做复杂任务时，常见的做法是把子 Agent 当成一个“更聪明的工具”：主会话派活，子 Agent 独立跑工具调用、MCP 查询、文件处理，最后把结果带回。但实际跑起来，主会话经常被污染：子 Agent 的中间推理、工具返回原文、调试日志、甚至 MCP 返回的 HTML 片段混进主上下文，导致后续推理漂移，上下文窗口快速耗尽。

这还不是最麻烦的。更危险的是，子 Agent 如果共享主会话的 memory 或工具权限，一次失控的工具调用可能覆盖主会话的状态，或者把外部注入的内容带回主决策链路。

## 问题

session 隔离不等于简单地把消息列表分开。实际要处理三个层面：

1. **上下文污染**：子 Agent 的思考链和工具输出被当作普通消息写回主会话。
2. **状态串扰**：两边共享 memory、环境变量或 MCP 连接，子 Agent 的副作用影响主会话。
3. **权限越界**：子 Agent 拥有主会话同等级的工具权限，可以访问敏感数据或执行危险操作。

在 OpenClaw 中，如果你只给子 Agent 开一个新的 session id，但继续沿用主会话的 memory backend、插件状态或事件总线，隔离基本是无效的。

## 做法/步骤

我目前用的方式是把子 Agent 包成一个“黑盒函数”，主会话只看到输入和结构化输出。

### 1. 独立 session + 独立 memory namespace

创建子 Agent 时不要 fork 主会话的 conversation id，而是显式创建独立 session，并设置单独的 memory namespace。示意配置：

```yaml
subagent_session:
  isolated: true
  parent_session_id: main-session-01
  memory_namespace: subagent/task-parse-log
  ttl: 300
  max_steps: 20
```

这样即使子 Agent 调用 memory 工具，写的也是子命名空间，主会话读不到，反之亦然。

### 2. 强制结构化返回，不做文本转发

子 Agent 的 system prompt 明确要求只返回 JSON 摘要，不附带工具原始输出：

```text
You are a subagent. Return a JSON object with keys:
result, confidence, artifacts, error.
Do not include raw tool output, logs, or intermediate reasoning.
```

在 OpenClaw 侧，我会再包一层 `post_process`：把子 Agent 的最终消息解析成 JSON，校验 schema，只把摘要写入主会话；如果子 Agent 返回了非 JSON 或超长内容，直接截断并标记为 `low_confidence`。

### 3. 工具白名单和最小权限

给子 Agent 分配单独的 MCP 配置，只暴露必需工具。比如主会话有 `file_write`、`db_query`、`slack_send`，子 Agent 只给 `file_read` 和 `log_parse`。不要在子 Agent 的 MCP server 里直接复用主会话的 token 或 scope。

### 4. 显式传参，不靠全局状态

子 Agent 需要的上下文，在主会话调用时打包成 JSON 传入，不让它去读主会话的 memory 或环境变量。这样可以避免子 Agent 拿到不必要的历史信息，也方便测试和回放。

### 5. 设置 TTL 和 step 上限

子 Agent 跑飞是最常见的污染源。限定最大步数和 session TTL，超时直接终止，避免它反复工具调用把大量中间结果累积进上下文。

## 踩坑点

- **共享 memory backend**：只改 session id 不改 memory namespace，等于只隔离了消息表，没隔离状态。
- **把异常堆栈当正常返回**：子 Agent 崩溃时，有些封装会把 traceback 原样带回主会话，不但污染上下文，还可能泄露路径信息。一定要在 wrapper 里捕获异常，只返回一个标准错误码。
- **MCP 输出未脱敏**：日志工具返回几百行原始日志，子 Agent 又原样转发。应在子 Agent 输出前做长度截断和敏感信息过滤。
- **事件流未隔离**：如果子 Agent 事件直接发布到主会话事件总线，主会话的 UI 和插件会收到大量子 Agent 中间事件，干扰判断。
- **误用 `send_message`**：子 Agent 内部如果调用主会话的 `send_message` 接口而不是通过返回值传递，会导致消息顺序和归属错乱。

## 可复用建议

1. **封装成标准子 Agent 模板**：不要在业务代码里每次手写隔离逻辑。做一个 `run_isolated_subagent(task, tools, ttl)` 函数，统一处理 session 创建、memory namespace、返回解析和异常兜底。
2. **主会话只保留摘要缓存**：对相同参数的子 Agent 调用做短期缓存，避免重复拉取。
3. **用标记区分主/子消息**：在主会话上下文中，子 Agent 的返回统一加上 `[subagent_summary]` 前缀或元数据标记，方便后续审计和过滤。
4. **CI 检查**：写一个简单的测试，跑完主会话任务后扫描上下文，确认没有出现子 Agent 的工具调用 trace 或原始输出，只有摘要。
5. **日志和 trace 分离**：子 Agent 的详细日志进入独立存储，不要和主会话的 debug 日志混在一起。

## 总结

session 隔离不是“多开几个 session”那么简单。对子 Agent 来说，主会话应该是一个干净的调用方，子 Agent 是一个有明确边界的黑盒函数。控制输入、输出、工具权限、memory 命名空间和事件流，才能真正做到子 Agent 不污染主会话。否则隔离只是表面上的，崩溃或注入迟早会发生。

如果现在你的 OpenClaw 主会话里已经混入了子 Agent 的中间输出，先把隔离边界补上，再考虑清理上下文——不然清理完还会继续污染。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/d3cf54fa68296f9a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/5bf434624ab2491a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/450265b480771df6.png)

