---
title: OpenClaw 的 session 隔离：给子 Agent 划清上下文边界
feedId: 34187
source: 综合讨论
publishedAt: 2026-08-22
---

## 背景

在 OpenClaw 里跑长任务时，常见做法是把某个子任务拆给子 Agent：比如主 Agent 负责规划，子 Agent 去抓取页面、整理文件、调用 MCP 工具。默认情况下，如果子 Agent 直接继承父 session，或者桥接工具把子 Agent 的执行轨迹原样带回，主会话会很快被污染。

典型表现是：主 Agent 开始重复分析子任务里的临时路径；突然引用某个只在子任务里出现过的变量；或者把子 Agent 的重试堆栈当成新任务继续处理。最后上下文窗口被大量中间输出占满，主任务规划能力下降。

## 问题本质

OpenClaw 的 session 本质上是上下文容器，不是线程。子 Agent 如果往父 session 写消息，这些消息就会参与主 Agent 的下一步推理。所谓“污染”，不是子 Agent 做错了，而是边界没有显式建立。

真正需要的隔离有三层：

1. 上下文隔离：子 Agent 看不到不必要的父上下文，父 Agent 也看不到子过程。
2. 工具/资源隔离：子 Agent 使用独立工作区，避免读写竞态。
3. 返回契约隔离：主会话只接收最终结构化结果，不接收过程日志。

## 做法/步骤

### 1. 子 Agent 使用独立 session

不要让子 Agent 复用父 session_id。使用 fork 或新建 session，并关闭继承。

```yaml
subagent:
  session:
    inherit: false
    scope: "child"
    ttl: 1800
  return:
    mode: "final_only"
    schema: "json"
    strip_tool_output: true
  limits:
    max_turns: 12
    max_output_tokens: 1200
```

伪代码示例：

```python
def run_child(parent_id, task_brief):
    child = openclaw.sessions.fork(parent_id, inherit=False)
    result = openclaw.agents.run(
        session_id=child.id,
        prompt=build_prompt(task_brief),
        return_mode="final_json",
    )
    return sanitize(result.summary)
```

关键点：`inherit=False`、`return_mode="final_json"`。子 session 的中间消息留在子 session，不写回父级。

### 2. 通过 bridge 工具返回 sanitized 结果

主 Agent 不应直接调用子 Agent 的底层接口。封装一个 bridge 工具，内部完成子 Agent 运行、等待结束、抽取 final 消息、过滤字段，最后只返回摘要。

```python
def bridge_subagent(task_brief):
    result = run_child(main_session_id, task_brief)
    return {
        "status": result.status,
        "summary": result.summary[:800],
        "artifacts": result.artifacts,
        "blockers": result.blockers,
    }
```

不要在 bridge 里返回 `result.messages` 或 `result.trace`。

### 3. 给子 Agent 独立工作区

如果子任务需要文件系统或浏览器，给每个子 session 分配临时目录或命名空间。不要让多个子 Agent 共用 `/workspace/scratch`，否则会出现竞态和脏读。

```python
child_workspace = f"/tmp/openclaw/{child.id}"
```

MCP 工具调用也尽量限制在这个目录下。对只读数据可以共享，但写入必须隔离。

### 4. 明确返回契约

在子 Agent 的 prompt 里强约束输出格式，例如：

```text
只返回以下 JSON，不要返回过程日志：
{
  "status": "done|blocked|failed",
  "summary": "一段不超过200字的结果说明",
  "artifacts": ["文件路径或资源ID"],
  "blockers": ["阻塞项"]
}
```

主会话拿到后，再决定是否读取 artifacts。

## 踩坑点

- **复用 session_id**：等于没有隔离，子 Agent 的每条消息都会进入主上下文。
- **bridge 返回全部消息**：有些实现会直接返回底层 LLM 的 messages 数组，这是污染重灾区。
- **返回全量 JSON dump**：即使开了 final_only，如果 final 里塞了完整数据，主会话仍会被撑爆。
- **异常 trace 直推父会话**：子 Agent 报错时，直接把堆栈返回给主 Agent，主 Agent 会试图修复子 Agent 内部问题，结果越跑越偏。应只返回错误码和一行原因。
- **工作区共用**：并行子 Agent 写同一路径，产生竞态，最终返给主会话的是损坏文件。

## 可复用建议

- 封装唯一的 `run_child` helper，禁止裸调子 Agent 接口。
- bridge 返回前做后处理：截断、过滤路径、移除敏感字段。
- 对子 Agent 输出做 schema 校验，不合格就重跑或降级。
- 监控父 session 的消息类型分布：如果 tool 消息占比异常升高，说明隔离失效。
- 给子 Agent 设置独立 token 预算，主会话只扣减 bridge 返回的 token。

## 总结

子 Agent 污染主会话，通常不是能力问题，而是边界问题。独立 session、只回传 final payload、隔离工作区、统一 bridge 封装，这四件事做完，主 Agent 才能保持稳定规划，子 Agent 也能在受限上下文里高效执行。对于 OpenClaw 上的多步自动化和 MCP 组合任务，这套隔离策略可以复用。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/7761c8400d5f8b58.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/9ac8803f51ebce23.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/7db0811d1fd38590.png)

