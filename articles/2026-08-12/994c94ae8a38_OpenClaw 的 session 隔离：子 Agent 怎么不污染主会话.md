---
title: OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话
feedId: 32739
source: 综合讨论
publishedAt: 2026-08-12
---

## 背景：子 Agent 调用中的上下文“串味”

在 OpenClaw 里拆分任务时，我们习惯把独立功能封装为子 Agent，由主 Agent 按需调用。这样做逻辑清晰，但很快会踩到一个坑：子 Agent 的执行过程、中间思考甚至内部格式数据会回流到主会话，导致主 Agent 后续判断被污染。

例如，一个用于查数据库的子 Agent 返回了详细的 SQL 执行计划与临时表名，主 Agent 在下一轮对话中可能会错误地将这些 SQL 片段当作业务上下文，产生虚构操作。又或者，子 Agent 无意间修改了全局缓存，让看似无关的任务之间产生数据串扰。这些问题背后的本质是**缺乏 session 隔离**。

## 问题分析：污染的具体表现

污染通常有三种形式：

1. **上下文膨胀**：子 Agent 的完整 reasoning trace 冲进主会话，挤占 token 窗口，主 Agent 的注意力被噪声拉低。
2. **记忆串扰**：OpenClaw 的自动记忆（如 Long-Term Memory 插件）可能把子 Agent 的对话历史合并回主记忆库，造成长期幻觉。
3. **状态冲突**：若子 Agent 与主 Agent 共享某些 MCP 工具的实例（例如同一个浏览器 tab、同一个文件句柄），操作会互相覆盖，导致状态错乱。

典型症状：子 Agent 调用结束后，主 Agent 突然开始提及子领域的术语；或者在后续轮次中，本应引用主任务变量时，却冒出了子 Agent 的临时变量名。

## 做法：三步实现可靠隔离

### 1. 强制分配独立 session_id

OpenClaw 允许在调用子 Agent 时传入自定义的 `session_id`。这是最基础的隔离手段：

```python
# 错误示范：沿用主 session
result = child_agent.run(task, session_id=main_session_id)

# 正确做法：基于主 session 派生独立子 session
sub_session = f"{main_session_id}_sub_{task_id}"
result = child_agent.run(task, session_id=sub_session)
```

这确保了子 Agent 的对话上下文、内置缓存都绑定在自己的 session 生命周期内。可在 `run` 方法的 `options` 中额外配置 `isolated_memory=True`，阻止子 session 的对话记入主记忆流。

### 2. 裁剪返回内容，只透传必要数据

不要直接把子 Agent 的最终回复怼回主 Agent 的上下文。在调用后增加一个提取层，只保留结构化结论：

```python
raw = child_agent.run(task, ...)
# 裁剪：只取摘要字段，丢弃中间推理
clean_output = {
    "status": raw.get("status"),
    "summary": raw.get("final_answer"),
}
main_context.add("last_sub_result", json.dumps(clean_output))
```

如果使用了 OpenClaw 的 Workflow 组件，可以在“子流程”节点的输出映射中手动指定 `output_mapping`，只映射关键字段，让噪声留在子流程内部。

### 3. 工具实例级隔离

当子 Agent 需要使用 MCP tools（例如 `browser_navigate`、`memory_search`）时，**避免与主业务共享同一 tool 实例的持久状态**。

建议为子 Agent 创建独立的 tool 适配器，或在工具调用时明确指定命名空间：

```python
tool = mcp_client.get_tool("vector_search", namespace=sub_session_id)
```

如果是浏览器自动化，可通过分页签或独立 browser context 实现隔离。在 Playwright 工具封装中，可维护一个 `context_id` → 浏览器实例的字典，每次子 Agent 调用时传入独有的 `context_id`，结束后主动关闭该 context，彻底清除残留。

## 踩坑点与应对

* **错误使用全局配置的 Agent 默认 session**  
  若在项目初始化时设置了 `default_session_id`，所有未显式传入 session 的调用会自动复用该 ID。排查之道：始终显式传递 session_id，并在部署检查中加入 lint 规则，禁止使用默认值。

* **多线程并发时的 key 冲突**  
  如果子 session_id 仅基于静态前缀生成（比如 `main_session_sub_1`），多个并行子任务会写入同一 session，造成交叉污染。解决方案是将时间戳或唯一请求 ID 嵌入子 session_id 生成函数。

* **长期记忆插件的静默回写**  
  部分记忆插件在每次对话结束后会自动将整段交互写入向量库，并默认使用全局索引。需在插件配置中开启 `per_session_index=True`，或在每次子完成后主动调用 `memory.purge(session_id=sub_session)`，防止积累无用痕迹。

* **子 Agent 内部的 log 输出混入返回值**  
  有时因为调试打印，子 Agent 的输出中包含了 `print` 语句的文本。这可以通过在 runner 中设置 `log_stream=SeparateBuffer()`，并将该缓冲区的数据单独路由到日志系统，而不是混入返回对象。

## 可复用建议

封装一个通用的 `IsolatedSubAgentRunner`，内部整合上述逻辑：

- 自动生成带唯一 ID 的子 session；
- 执行后默认裁剪，仅返回 `final_answer` 字段；
- 可选参数控制是否保留子 Agent 的完整上下文用于调试；
- 自动创建和销毁 MCP 工具命名空间；
- 在结束后确保子 session 数据从活动内存中移除。

将此 Runner 作为项目工具库的一部分，新加入的 Agent 调用统一走这个入口，可大幅降低排查成本。

另外，在 CI 中增加一个简单的冒烟测试：分别以两个不同子任务运行同一 Agent，随后检查主 session 的上下文，不应包含任何子任务特有的关键字。这是验证隔离是否健壮的工程化手段。

## 总结

OpenClaw 的 session 隔离不是单一开关，而是一层需要从 session_id、输出裁剪、工具实例三个维度共同构建的防护。只要把子 Agent 视为“无状态的函数调用”，强制输入明确、输出洁净，就能避免上下文污染和状态串扰。实际落地时，结合可复用的 Runner 封装和自动化检查，可以让多 Agent 协作既灵活又可控。

---

