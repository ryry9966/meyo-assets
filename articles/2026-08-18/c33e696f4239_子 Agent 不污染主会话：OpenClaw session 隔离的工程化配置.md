---
title: 子 Agent 不污染主会话：OpenClaw session 隔离的工程化配置
feedId: 33750
source: 综合讨论
publishedAt: 2026-08-18
---

## 背景

在 OpenClaw 里用子 Agent 做批量文件检索、代码审查或 MCP 数据抓取时，默认运行方式往往是把子 Agent 当作当前 session 的子任务。短期看没什么问题，但任务一多，主会话就会变得很难维护：子 Agent 的中间推理、工具调用日志、甚至临时 system prompt 都会混进主历史。

我们踩过几次之后，把子 Agent 改成显式隔离 session，只允许返回结构化结果。下面记录的是实际配置思路和排障点。

## 问题表现

1. **主上下文膨胀**：子 Agent 的 `tool_calls`、`tool_result`、多轮重试过程全部写回主 session，主 Agent 很快被无关消息淹没。
2. **指令漂移**：子 Agent 的临时 system prompt 或任务描述残留在共享上下文里，主 Agent 后续行为被带偏。
3. **MCP 命名空间冲突**：多个子 Agent 共用一个 MCP server 或工具前缀，状态互相覆盖。
4. **变量泄漏**：子 Agent 内设置的临时变量在主会话可见，造成误读。

## 做法/步骤

### 1. 给子 Agent 开独立 session

在子 Agent 定义中显式声明隔离模式。示例 `openclaw.yaml`：

```yaml
agents:
  file-reviewer:
    model: claude-sonnet
    session:
      mode: isolated
      max_turns: 12
      keep_messages: false
    tools:
      - read_file
      - grep
    return:
      format: json
      include_trace: false
```

这里的 `session.mode: isolated` 表示子 Agent 不共享主会话上下文。`keep_messages: false` 表示不保留中间消息，`include_trace: false` 表示不回传工具调用轨迹。

### 2. 限制返回协议

只允许子 Agent 返回结构化结果，主会话收到后只追加一条消息：

```json
{
  "status": "ok",
  "summary": "reviewed 12 files, 3 issues found",
  "files": ["src/a.ts", "src/b.ts"],
  "issues": [...]
}
```

不要直接返回自然语言的长篇解释。主 Agent 拿到 JSON 后自行决定如何压缩或展示。

### 3. MCP 工具做作用域隔离

如果子 Agent 需要 MCP，不要和主会话共用同一个 client。可以按作用域配置前缀：

```yaml
mcp:
  - name: code-index
    scope: file-reviewer
    prefix: fr_
  - name: code-index
    scope: main
    prefix: main_
```

这样即使两个作用域都连接同一个 MCP server，工具名也不会互相覆盖。

### 4. 显式传输入，不依赖全局变量

子 Agent 启动时，主 Agent 只传 `task` 和必要的 `context`，不要在共享 session 里放临时状态。wrapper 示例：

```python
def run_isolated(agent_name, input_payload):
    result = openclaw.run_agent(
        agent=agent_name,
        input=input_payload,
        session_mode="isolated",
        return_format="json"
    )
    # 只保留 result.summary 和 result.data，不追加完整轨迹
    return compact_result(result)
```

## 踩坑点

- **隔离后拿不到主历史**：开了 `isolated` 后，子 Agent 会丢失主会话上下文，容易重复问“这个路径是什么”。解决方法是把必要信息压缩成一个 `context` 字段传进去，而不是重新放开共享。
- **MCP 连接不自动关闭**：如果子 Agent 在 isolated session 中打开了 MCP 连接，任务结束后连接不会随 session 销毁。需要在 `finally` 里显式 close，否则会看到连接数持续增长。
- **返回结果里残留 tool_calls**：有些框架会把子 Agent 的最终输出包成 assistant message，里面可能还带着空的 `tool_calls` 字段。主 Agent 如果直接读取字符串，可能会解析错。建议在 wrapper 里做一次字段裁剪，只保留 `content` 或 `summary`。
- **插件依赖全局 session 状态**：部分 automation 插件假设 `ctx.session` 始终存在，在 isolated session 下会直接报错。需要改成通过参数或外部存储（SQLite/Redis）传递状态。

## 可复用建议

- 子 Agent 尽量做无状态任务，或只通过 MCP/外部存储读写。
- 用 JSON schema 校验子 Agent 输出，不合规就重试一次，避免脏数据进入主会话。
- 给隔离配置做一个模板，新子 Agent 默认继承，不要每次手写。
- 如果使用持久化 session 存储，部署脚本里加一个定时清理任务，例如 `openclaw session gc --older-than 7d`（按实际 CLI 调整），避免孤儿 session 堆积。

## 总结

session 隔离不是把子 Agent 关进黑盒，而是给它一个受控的临时上下文：只接收明确输入，只返回结构化结果，用完即弃。这样才能让主会话保持稳定，MCP 连接和插件状态也不容易串扰。对于我们目前的自动化任务量，这个配置基本能解决 80% 以上的上下文污染问题，剩下的就是约束子 Agent 不要过度自由发挥。

---

