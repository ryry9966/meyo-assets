---
title: OpenClaw 实战：给子 Agent 独立 Session，主会话不再被污染
feedId: 33215
source: 综合讨论
publishedAt: 2026-08-15
---

# OpenClaw 实战：给子 Agent 独立 Session，主会话不再被污染

## 背景

在 OpenClaw 里做复杂自动化时，主 Agent 经常要把“查资料、跑脚本、调 MCP、生成报告”这类任务拆给子 Agent。尤其是接了不少 MCP server 或插件后，主会话很容易变成调度中心。

但很多人的第一版实现是直接在主会话里调用子 Agent，或者让子 Agent 把中间过程写回主历史。跑起来之后就会发现：主 Agent 越来越“笨”，容易重复引用无关信息，token 消耗暴涨，甚至被某次子任务里的错误堆栈带偏。

这个问题的本质不是子 Agent 能力不行，而是 session 没有隔离。

## 问题表现

如果不做隔离，通常会出现以下现象：

1. **上下文污染**：子 Agent 的中间草稿、调试输出、工具调用日志全部进入主历史，主 Agent 后续推理时容易被噪音干扰。
2. **状态污染**：子 Agent 改了环境变量、写了临时文件、更新了某个全局配置，影响到主会话后续任务。
3. **指令漂移**：子 Agent 的 system prompt 或约束片段混进主 Agent 的决策上下文，导致主 Agent 行为偏离原始目标。
4. **清理不可控**：子任务异常退出后，session 没有关闭，残留的 session 对象和 memory 写入持续累积。

## 做法/步骤

核心原则：**子 Agent 关在独立 session 里，只开一扇窄门返回结构化结果。**

### 1. 为每个子任务创建独立 session

不要让子 Agent 直接使用主会话的 `complete` 或 `ask`。在 OpenClaw 里使用 `spawn` 创建独立子 session：

```python
import openclaw

def run_isolated_task(task: str, ctx: dict, timeout: int = 300):
    child = openclaw.spawn(
        name="sub_research",
        system="You are a research subagent. Return JSON only.",
        session={
            "scope": "ephemeral",   # 临时 session，不写主历史
            "ttl": timeout,
        },
        memory={"namespace": f"tmp:research:{ctx.get('task_id')}"},
    )
    try:
        result = child.run(task, context=ctx)
        return {
            "ok": result.ok,
            "summary": result.summary[:2000],
            "artifacts": result.artifacts,
            "error": result.error,
        }
    finally:
        child.close()
```

这里的关键点：子 session 是 `ephemeral`，任务结束就销毁；memory 使用独立 namespace，不写入主记忆。

### 2. 最小化入参

不要图省事把整个主历史传给子 Agent。只传任务描述、必要上下文和约束。主 Agent 可以在调用前自己压缩上下文，或者只拷最近 N 轮关键信息。

### 3. 只返回结构化结果

子 Agent 的输出不要直接拼回主会话。主会话只读取 `summary`、`artifacts` 引用和 `error`。如果子 Agent 生成了大文件，只把文件路径或对象存储 key 带回主会话，不要内联大文本。

### 4. 配置默认隔离策略

如果你有多个插件或自动化流程都会调用子 Agent，建议在 `openclaw.yaml` 里配置默认行为：

```yaml
subagent:
  default_session_mode: isolated
  max_context_tokens: 4000
  return_mode: summary_only
  auto_close: true
```

这样即使某个插件忘记手动清理，也不会把子 Agent 的原始输出直接灌进主会话。

### 5. 异常路径也要清理

`try/finally` 是最低要求。实际工程里还要考虑超时、取消、进程被杀等情况。可以在 OpenClaw 的任务调度外层加一个定时清理任务，定期关闭超过 TTL 仍未结束的 child session。

## 踩坑点

- **全局变量串味**：并发跑多个子 Agent 时，不要用全局变量或模块级状态传递子任务信息。用 `contextvars` 或显式参数。
- **artifact 内联膨胀**：子 Agent 返回一个 100KB 的文本文件，主会话直接存进 history，token 直接爆炸。只存引用。
- **嵌套子 Agent 形成 session 树**：子 Agent 又调用子 Agent，层级没限制，最后 session 树深不可测。建议设置 `max_depth`，超过就禁止继续 spawn。
- **MCP/插件认证丢失**：子 session 隔离后可能拿不到主会话的认证信息或 scope token。需要显式传递，或使用短期 token。
- **清理失败累积**：某些异常路径下 `child.close()` 也可能失败。要有兜底过期机制，不要只依赖 `finally`。

## 可复用建议

如果你不想每次都手写隔离逻辑，建议封装一个统一的 `spawn_subagent` 工具，所有插件、MCP 调用、自动化流程都走这个入口。工具内部负责：

- 创建独立 session
- 注入最小上下文
- 校验返回 schema
- 截断 summary
- 记录 child session id 到日志
- 统一关闭和清理

这样主 Agent 只需要调用一个工具，而不是直接面对子 session 的生命周期。

另外，对于长任务，建议让子 Agent 写文件或对象存储，通过 artifact 引用回传检查点，而不是把中间状态写进主上下文。

## 总结

子 Agent 不污染主会话的核心，不是“禁止子 Agent 说话”，而是把它放在独立 session 里，只允许通过结构化结果返回。控制进出，比事后清理更可靠。

最小落地就三件事：**独立 session、最小入参、结构化返回**。做完这三件事，主会话的稳定性和 token 成本会有明显改善。

---

