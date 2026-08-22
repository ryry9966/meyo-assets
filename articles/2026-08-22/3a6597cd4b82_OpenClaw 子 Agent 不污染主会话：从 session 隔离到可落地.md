---
title: OpenClaw 子 Agent 不污染主会话：从 session 隔离到可落地的调用契约
feedId: 34155
source: 综合讨论
publishedAt: 2026-08-22
---

# OpenClaw 子 Agent 不污染主会话：从 session 隔离到可落地的调用契约

## 背景

在 OpenClaw 里跑复杂任务时，主 Agent 经常会 spawn 子 Agent 去处理子任务：读文件、调 MCP 工具、做批量自动化、跑插件逻辑。默认情况下，很多实现会直接复用当前 session，或者把子 Agent 的完整 transcript 拼回主上下文。结果就是：子 Agent 的中间过程、工具日志、错误堆栈、重复输出不断进入主会话，轻则 token 膨胀、注意力漂移，重则工具状态串线、敏感信息残留，甚至影响后续决策。

这里的核心不是“子 Agent 不要输出”，而是要控制**什么内容能回到主会话**。session 隔离的目的，是让子 Agent 在自己的边界内干活，主 Agent 只拿到它需要的结论。

## 问题

我们实际遇到的污染主要体现在三类：

1. **上下文污染**：子 Agent 的每一步思考、每一次工具调用结果，都被写入主 session 的消息列表。主 Agent 后面做判断时，眼前全是子任务的碎片信息。
2. **状态污染**：子 Agent 复用主 session 的 MCP 连接或插件状态，比如数据库游标、文件句柄、环境变量，导致后续操作互相干扰。
3. **越权/串线**：子 Agent 拿到了主会话的完整 memory，误用用户偏好、路径或凭证信息，做出超出子任务范围的调用。

这些问题的共同点是：没有显式定义 session 边界。

## 做法/步骤

### 1. 给子 Agent 一个独立 session，不要继承主上下文

创建子 Agent 时，显式指定新的 `session_id`，并且不要默认继承主 session 的 `messages`。如果 OpenClaw 配置里允许，设置类似：

```yaml
subagent:
  session:
    inherit_parent: false
    context_mode: minimal  # 可选 none / minimal / summary
```

这样做的目的是切断“自动继承”，让子 Agent 从干净上下文开始。

### 2. 输入最小化：只传任务和必要白名单字段

不要把整个主对话历史传给子 Agent。定义一个固定输入结构，例如：

```json
{
  "task": "读取 ./data/orders.csv 并统计今日订单数",
  "context": {
    "workspace": "/home/user/project",
    "date": "2026-04-28"
  },
  "return_schema": {
    "order_count": "integer",
    "error": "string|null"
  }
}
```

主 Agent 只注入子任务真正需要的信息。用户偏好、历史决策、无关工具结果都不放进 `context`。

### 3. 输出摘要化：只回传结构化结果或 summary

子 Agent 完成后，不要直接把它的完整日志写回主 session。在调用层面加一层包装，只保留 `result` 和 `summary` 字段。伪代码示例：

```python
def run_subagent(task, context):
    sub = openclaw.spawn_subagent(
        session_id=generate_id(),
        context_mode="minimal",
        return_mode="summary"
    )
    result = sub.run(task, context)
    # 丢弃 result.transcript，只保留结构化返回
    return {"task": task, "summary": result.summary, "result": result.data}
```

`return_mode` 如果支持 `summary`，就优先用；不支持就在 wrapper 里手动截断，保留最后的结构化输出和一行总结。

### 4. 独立 MCP session，避免工具状态串线

如果子 Agent 要调用 MCP 工具，给子 Agent 创建独立的 MCP client，而不是直接复用主会话的连接。对于有状态工具（数据库、文件系统、浏览器），这一点尤其重要。可以在子 Agent 的配置里单独声明 MCP server：

```yaml
subagent:
  mcp:
    servers:
      - name: "filesystem"
        isolated: true
```

如果 OpenClaw 版本不支持细粒度配置，至少要在子 Agent 运行结束后显式关闭它的 MCP 连接，释放游标和句柄。

## 踩坑点

- **子 Agent 输出默认写回主 session**：很多 wrapper 会静默把子任务的全部输出 append 到主消息列表。即使主 Agent 没用这些内容，它们也已经占用了上下文窗口。必须显式设置 `return_mode` 或手动裁剪。
- **插件自动注入主上下文**：有些插件为了“方便”，会自动把当前主 session 的历史塞进子 Agent 的 prompt。这会让隔离形同虚设，需要检查插件配置，关闭自动注入。
- **独立 session 但没有同步关键环境**：隔离做得太彻底，子 Agent 连工作目录、Python 环境、API key 都没有，反而无法执行任务。解决方法是只同步白名单环境变量，而不是整个 memory。
- **MCP 连接复用导致状态污染**：比如主 Agent 打开了一个数据库事务，子 Agent 复用同一连接后提交或回滚，影响主流程。一定要让子 Agent 使用独立连接。
- **日志误写回**：子 Agent 的错误堆栈可能通过异常处理被主 session 捕获并写回。建议在 wrapper 里 catch 异常，只返回 `{"error": str(e)}`，不要把 traceback 带到主上下文。
- **主 session 的 token 增长没有监控**：隔离前后没有量化指标，出了问题才发现。建议在每次子 Agent 调用前后记录主 session 的 token 数或消息数。

## 可复用建议

1. **定义子 Agent 调用契约**：固定输入 JSON 结构，字段白名单，可选 `return_schema`。这样隔离边界清晰，也方便后续做 schema 校验。
2. **做一个统一的 `run_isolated_subagent` wrapper**：所有子 Agent 调用都走这个入口，禁止直接 `spawn_subagent`。在 wrapper 里处理 session 创建、日志丢弃、异常转换、MCP 清理。
3. **定期审计主 session 的 token 分布**：可以做一个轻量工具，统计主 session 里来自子 Agent 的 token 占比。正常情况下应该很低，异常时告警。
4. **敏感操作单独开 session**：例如文件写入、网络请求、数据库变更，即使子 Agent 失败，也不应污染主会话的判断链。隔离 session 便于事后审计。
5. **输出 schema 化，便于机器消费**：让子 Agent 返回 JSON，而不是自然语言段落。主 Agent 可以直接解析，不需要再花 token 去理解自然语言结果。

## 总结

session 隔离不是“不让子 Agent 说话”，而是通过三个动作把边界钉死：**输入最小化、输出摘要化、状态独立化**。在 OpenClaw 里落地时，优先从 wrapper 和配置入手，而不是靠提示词约束子 Agent。真正的隔离必须体现在代码路径和 session 对象上，否则只是自我安慰。

把子 Agent 当成一个远程服务来设计：你给它一个请求，它给你一个响应，中间过程你不关心，也不该进入你的主上下文。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/ef7bc32c38b7ffcd.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/68ff0fd97615d33e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/7134b1a303a6bd59.png)

