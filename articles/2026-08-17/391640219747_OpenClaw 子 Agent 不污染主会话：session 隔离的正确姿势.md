---
title: OpenClaw 子 Agent 不污染主会话：session 隔离的正确姿势
feedId: 33539
source: 综合讨论
publishedAt: 2026-08-17
---

## 背景

在 OpenClaw 里，子 Agent 经常被用来执行长任务：批量读文件、跑脚本、抓网页、生成报告。默认情况下，子 Agent 多半会继承主会话上下文，并把工具调用、中间推理、错误重试信息一路回传。主 Agent 很快就会遇到三个问题：

1. 上下文窗口被大量中间步骤占满；
2. 主会话把子任务的临时假设当成主线结论；
3. 子 Agent 写入的 memory 或状态污染后续任务。

这本质不是模型能力问题，而是 session 边界没有划清楚。子 Agent 应该更像独立进程：你给它输入，它给你最终结果和摘要，而不是像一个群聊成员不断插话。

## 问题表现

实践中，最明显的症状是：

- 主 Agent 开始引用子任务里的局部变量、错误信息或未验证结论；
- 主会话上下文中出现大量 `tool_call`、`tool_result`、重试栈；
- 子 Agent 的调试日志进入主 memory，后续任务读取后行为异常；
- 上下文长度暴涨，主 Agent 开始丢失早期关键指令。

如果每次都靠人工截断上下文或重启会话，工程上不可持续。

## 做法 / 步骤

在 OpenClaw 中做 session 隔离，核心不是单一开关，而是四层隔离：**上下文隔离、返回结构隔离、工具权限隔离、内存命名空间隔离**。

以一个 `file-audit` 子 Agent 为例，推荐配置如下：

```yaml
subagent:
  name: "file-audit"
  isolated_session: true
  context_mode: "summary"
  memory_namespace: "subagent/file-audit/{{run_id}}"
  return_fields: ["final_summary", "artifacts"]
  tools:
    allow: ["read_file", "grep", "list_dir"]
    deny: ["update_memory", "send_to_main", "write_file"]
  max_context_tokens: 6000
```

具体拆开看：

### 1. 建独立 session，不要复用主 session id

创建子 Agent 时，明确分配新的 `session_id`，例如 `main-{{parent_id}}-sub-{{task_id}}`。不要让它直接跑在主 session 的同一上下文中。

这一步能避免子 Agent 的中间 tool call 直接进入主会话历史。即使子 Agent 内部发生多次重试，也只会留在自己的 session 里。

### 2. 返回结构只保留最终摘要

把子 Agent 的回传模式设为 `summary`，而不是 `full_history`。让它只返回：

```json
{
  "final_summary": "...",
  "artifacts": ["s3://.../report.md"],
  "status": "ok"
}
```

中间步骤、错误堆栈、调试输出都不要回传主会话。大块数据用 artifact 文件或对象存储传递，不塞进对话。

### 3. 工具白名单，禁止写主会话

子 Agent 只给必要工具。关键是要禁止两类：

- 写主 memory 的工具，例如 `update_memory`；
- 直接向主会话发消息的工具，例如 `send_to_main`。

否则即使 session 隔离了，子 Agent 仍可能通过这些工具把脏状态写回主线。

### 4. 使用独立 memory namespace

OpenClaw 的 memory 一般有命名空间。给子 Agent 分配 `subagent/{task_id}` 这种独立 namespace，避免写入默认的 `main`。

任务结束后，可以选择把子 namespace 中的高价值结论，通过人工审核或规则过滤后，再合并回主 memory。

### 5. 主会话只做落库和审阅

主 Agent 拿到子 Agent 返回的 `final_summary` 后，只做两件事：

- 校验摘要结构是否完整；
- 决定是否把摘要写入主 memory 或作为 artifact 保存。

不要直接把子 Agent 的完整输出追加到主上下文。

## 踩坑点

实际落地时，有几个容易被忽略的地方：

- **事件流泄漏**：进度事件、日志事件没有过滤，仍然进入主 session。隔离时要同时配置事件过滤器，只允许 `final_result` 类型事件回传。
- **MCP 工具回调绕过**：子 Agent 通过 MCP 调用主会话侧的工具时，隔离策略可能失效。需要检查 MCP tool 的调用来源，限制子 Agent 的 MCP 权限。
- **memory namespace 写错**：子 Agent 的 memory 配置没有生效，仍然写到 `main`。验证时可以直接查询主 memory 是否新增了子任务中间键。
- **子 session 长期不清理**：大量子 session 堆积，占用存储并影响后续检索。建议设置 TTL，或在任务结束后显式关闭。
- **系统提示继承过多**：主会话的系统提示被完整复制到子 Agent，可能泄漏敏感上下文，也浪费 token。子 Agent 只应保留任务必需的指令。

## 可复用建议

在实际工程里，可以把子 Agent 封装成一个工厂函数：

```python
def spawn_isolated_subagent(task_name, task_input):
    return {
        "name": task_name,
        "session_id": f"sub-{task_name}-{uuid4().hex[:8]}",
        "isolated_session": True,
        "context_mode": "summary",
        "memory_namespace": f"subagent/{task_name}/{{run_id}}",
        "return_fields": ["final_summary", "artifacts"],
        "tools": get_task_tools(task_name),
        "max_context_tokens": 6000,
        "ttl_seconds": 3600,
    }
```

这样避免每次手写配置，也降低漏配概率。同时建议：

- 为子 Agent 的输出定义 schema，便于主 Agent 校验；
- 关键任务在 CI 中测试隔离效果，确认主 session 不会出现子任务中间步骤；
- 定期审计主 memory，发现异常键时追溯来源。

## 总结

子 Agent 污染主会话，通常不是单点问题，而是上下文、返回结构、工具权限、内存命名空间四个层面同时失守。只开 `isolated_session` 并不能解决所有问题。务实的做法是：独立 session 跑任务，只回摘要，限制工具写回，使用独立 memory namespace，并在主会话侧做最后审阅。

这样，子 Agent 才能成为真正可编排的执行单元，而不是主会话里的噪声源。

---

