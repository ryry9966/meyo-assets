---
title: OpenClaw 子 Agent 污染主会话的排查与 session 隔离实践
feedId: 32849
source: 综合讨论
publishedAt: 2026-08-13
---

## 背景

在 OpenClaw 里用子 Agent 处理多步骤任务时，主 Agent 负责拆解和调度，子 Agent 负责具体执行：查数据、跑 MCP 工具、生成代码、回传结果。这套模式在自动化流水线里很常见。

但最近在跑一个长时间任务时，发现主会话上下文膨胀得异常快：主 Agent 原本只维护任务状态和调度信息，token 消耗却线性增长，几次交互后模型开始“忘事”，甚至把子 Agent 的调试输出当成用户输入来回复。

## 问题排查

打开 OpenClaw 的 session 记录和日志，定位到几个典型的污染来源：

1. 子 Agent 的中间推理步骤被写回主会话，包括多轮 `thought`、工具调用前的 planning。
2. 子 Agent 调用 MCP 工具时，工具返回的原始 JSON/HTML 原样进入了主上下文。
3. 子 Agent 异常时，堆栈信息和错误输出直接追加到父 session。
4. 部分插件在子 Agent 内部打印日志，通过共享的 stdout 或 logging handler 混入主会话。

根因是：子 Agent 默认继承了父 session 的 context 和返回通道，执行过程中的全部轨迹都被当成“有用信息”回传。主 Agent 的上下文窗口被噪声占满，有效信息密度迅速下降。

## 做法/步骤

### 1. 在子 Agent 定义中启用 session 隔离

在 OpenClaw 的子 Agent 配置里，显式关闭上下文继承，并限制返回模式。示例配置：

```yaml
agent:
  name: data-fetcher
  type: subagent
  session:
    isolate: true
    inherit_context: false
    return_mode: final_only
  mcp:
    namespace: "sub/fetcher"
    reuse_parent_session: false
```

关键字段：
- `isolate: true`：子 Agent 使用独立 session，不直接写入父 session。
- `inherit_context: false`：不自动继承父上下文，强制主 Agent 显式传参。
- `return_mode: final_only`：只返回最终结果，不返回中间步骤。

### 2. 显式传递必要的输入

隔离后，子 Agent 看不到主会话历史。主 Agent 在调用子 Agent 时，需要构造一个干净的 `input` 对象：

```python
result = subagent.run(
    name="data-fetcher",
    input={
        "task": "查询指定账号最近7天订单",
        "params": {"account_id": "A123", "days": 7},
        "context": "仅需要订单总数和总金额"
    }
)
```

不要偷懒传整个主会话的 messages 列表，否则隔离形同虚设。

### 3. 对子 Agent 输出做截断/摘要

即使只返回 final answer，子 Agent 也可能返回超长内容。可以在主 Agent 侧增加后处理：

```python
def summarize_sub_result(raw):
    # 限制最大 token，超出则用本地摘要
    if estimate_tokens(raw) > 800:
        return compact(raw)
    return raw
```

### 4. 监控主 session 的 token 增长

在主 Agent 的循环里加入上下文用量检查：

```python
if session.token_usage > MAX_MAIN_SESSION_TOKENS:
    compact_session()
```

阈值一般设为主上下文窗口的 60% 左右，给后续交互留出空间。

## 踩坑点

- **只开隔离，不限制返回**：子 Agent 虽然不写中间步骤，但可能把整个执行历史打包进 final answer 返回。务必设置 `return_mode: final_only`，并在主 Agent 侧再做一次截断。
- **MCP 工具的 stderr 绕过隔离**：有些 MCP 连接会把工具进程的 stderr 打到父进程日志。即使 session 隔离了，这些日志可能污染主 Agent 的日志上下文。建议单独配置 MCP 的日志输出，不要与主会话日志混用。
- **共享外部存储隐式污染**：子 Agent 如果直接往 Redis/文件里写数据，主 Agent 读取后可能拿到非预期的大段内容。为子 Agent 分配独立的 key 前缀或临时目录，并约定清理机制。
- **异常堆栈信息泄露**：子 Agent 报错时，不要把完整 stack trace 返回给主 Agent。截取关键错误信息即可，否则会污染上下文，还可能泄露内部路径。
- **配置继承陷阱**：有些团队会为了方便，在主配置里设 `session.inherit = true`，子 Agent 若不覆盖，隔离会失效。建议在子 Agent 模板里强制覆盖为 `false`。

## 可复用建议

1. 为每个子 Agent 定义明确的 I/O schema，输入只收结构化字段，输出只收 `{ "ok": true, "data": ... }` 或错误对象。
2. 子 Agent 的 session id 使用前缀区分，例如 `sub:data-fetcher:<task_id>`，便于排查和清理。
3. 在团队文档里约定：子 Agent 默认不共享主上下文，所有必要信息必须通过 `input` 传入。
4. 在 CI 或集成测试里加一个断言：跑完一组子 Agent 任务后，主 session 的 token 增长量不超过预设值。
5. 定期清理过期的子 Agent session，避免孤儿 session 占用内存。

## 总结

OpenClaw 的 session 隔离不是简单打开一个开关就能解决。它要求你同时管理好子 Agent 的输入边界、返回格式、日志通道和外部存储行为。核心原则是：**子 Agent 只应该从主 Agent 拿到完成当前任务所需的最小信息，并且只返回主 Agent 做决策所需的最小结果。** 这样主会话才能保持干净，长任务里的模型才不会越跑越“糊涂”。

---

