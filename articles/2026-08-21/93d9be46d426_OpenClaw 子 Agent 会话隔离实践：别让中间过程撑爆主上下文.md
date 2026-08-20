---
title: OpenClaw 子 Agent 会话隔离实践：别让中间过程撑爆主上下文
feedId: 33957
source: 综合讨论
publishedAt: 2026-08-21
---

## 背景

在 OpenClaw 里接 MCP、插件或自动化任务时，主 Agent 经常会把“搜索关键词、重试了 3 次、第 2 次工具返回超长、第 3 次终于拿到结果”这类中间过程全部带回主会话。主会话上下文很快膨胀，后续规划开始漂移，甚至把子 Agent 的报错当成事实继续推理。

工程上更合理的做法是：子 Agent 跑在独立 session 里，主会话只拿终态结果、结构化摘要和可回溯的 trace id。

## 问题

默认情况下，如果你只是简单 spawn 一个子 Agent，或直接调用 MCP 工具，回传内容往往包含：

- 多轮工具调用和中间输出
- 重复的 planning / action 文本
- 完整错误堆栈
- 与主任务无关的探索过程

这会带来几个实际问题：

1. **上下文污染**：主 Agent 后续规划会被无关中间文本干扰。
2. **token 成本失控**：一次子任务可能带回几万 token 的过渡内容。
3. **敏感信息泄漏进长期记忆**：子 Agent 读取的临时凭证、内部路径、调试日志被主会话记忆。
4. **排障成本高**：真正失败原因被淹没在中间过程里。

## 做法 / 步骤

### 1. 先判断哪些任务该隔离

不是所有子任务都需要隔离。适合隔离的场景：

- 多步浏览器 / 文件操作
- 批量检索、抓取、解析
- 代码生成后还带测试输出
- 调用有状态 MCP server
- 需要重试、回退的任务

简单只读查询、单步工具调用，没必要强隔离。

### 2. 启动子 Agent 时使用独立 session

示例配置片段，以你本地的 OpenClaw 版本为准：

```yaml
subagent:
  spawn:
    session_mode: isolated
    inherit_memory: false
    return_mode: final_only
    max_steps: 20
    max_output_tokens: 4000
```

关键点：

- `session_mode: isolated`：子 Agent 不直接写主会话历史。
- `inherit_memory: false`：避免把主会话长期记忆直接复制给子 Agent，也避免子 Agent 写回。
- `return_mode: final_only`：只返回最终消息，不返回中间步骤。
- `max_output_tokens`：给最终返回也设上限，防止摘要式返回被长 artifact 顶爆。

### 3. 定义返回契约

不要让子 Agent 自由发挥。主会话需要的是稳定、可解析的返回结构。建议用 JSON 或严格 Markdown 片段：

```json
{
  "status": "ok | error | timeout",
  "summary": "一句话结论",
  "result": "最终产出，必要时截断",
  "artifact_ref": "oss://bucket/path/result.json",
  "trace_id": "sub-20250407-1842-a1b2c3",
  "error_code": "OPTIONAL"
}
```

`result` 只放主 Agent 马上要用的内容。完整结果写外部存储，主会话只拿引用。

### 4. 通过 MCP / 插件封装

把上面的逻辑封装成主会话可调用的工具，而不是每次手动拼参数。例如：

```text
spawn_isolated_task(
  task: string,
  return_schema: json,
  artifact_backend: oss | local_fs,
  max_steps: int,
  timeout_seconds: int
)
```

工具内部创建隔离 session，等待子 Agent 完成，再把最终产物压缩成契约结构返回给主会话。

### 5. 原始日志外置

子 Agent 的完整 transcript、工具调用、中间输出，全部写入外部日志或对象存储。主会话只保留 `trace_id`。排障时根据 trace id 去外部日志系统查，而不是让主会话回忆。

## 踩坑点

### 1. 子 Agent 崩溃时不要回传完整 stacktrace

如果子 Agent 因工具异常、超时、权限不足失败，直接把完整错误堆栈回传主会话，会让主 Agent 误判成“任务失败原因”，甚至尝试去修复一个不属于自己职责的 bug。建议只回传：

```json
{
  "status": "error",
  "error_code": "BROWSER_TIMEOUT",
  "trace_id": "sub-..."
}
```

让主 Agent 知道“这个任务没完成，原因码是 XXX”，而不是阅读 300 行 Python traceback。

### 2. 隔离不等于干净

独立 session 只是上下文隔离，不是环境隔离。子 Agent 可能仍然继承主进程的环境变量、API keys、文件权限。需要显式做 env 白名单：

```yaml
subagent:
  env_allowlist:
    - HOME
    - TMPDIR
    - OSS_BUCKET
```

不要默认把所有环境变量透传。

### 3. 注意 MCP server 的状态

如果你的 MCP server 是有状态的，比如保持浏览器会话、数据库连接、内存缓存，子 Agent 即使开了独立 session，也可能和主 Agent 共享同一个 server 实例状态。要确认你用的 MCP server 是否 session-scoped。如果不是，最好为子 Agent 单独起一个 server 实例。

### 4. 调试时临时关隔离，会把主会话搞脏

排障时，你可能想“先别隔离，我看下完整过程”。这样主会话会被大量中间步骤污染。更安全的做法是：复制一个 debug session 去跑子任务，而不是在主会话里直接关隔离。

### 5. 小心孤儿 session

子 Agent 超时或异常退出时，如果外层没有正确回收，可能留下孤立 session 继续占用资源。务必在 wrapper 里加 `finally` 清理，并设置硬超时。

## 可复用建议

- **统一 task wrapper**：不要到处散落 spawn 逻辑，一个封装函数统一处理隔离、返回契约、日志外置。
- **限制子 Agent 步数和输出**：`max_steps`、`max_output_tokens` 必须设，否则一个失控子任务能炸掉整个上下文预算。
- **主会话只存摘要和引用**：完整 artifact 放对象存储，主会话只保留 `trace_id` 和一句话结论。
- **错误码比错误堆栈重要**：给常见失败定义 `error_code`，主 Agent 只需要分类型处理。
- **保留一条回到原始日志的路径**：可以是 trace id，也可以是日志文件路径，但必须能完整回溯。

## 总结

子 Agent 不污染主会话，核心不是“加一个开关”，而是把子任务当成一个有边界、有契约的异步执行单元：

1. 输入是明确指令；
2. 执行过程在独立 session；
3. 输出是结构化终态；
4. 完整日志外置；
5. 失败只返回可处理的错误码。

这样主 Agent 才能保持上下文干净，继续做它擅长的事——规划、判断、调度，而不是被中间过程牵着走。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/3d1fc3a7627d5109.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/20b106f568880c09.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/efd1ff259be72b05.png)

