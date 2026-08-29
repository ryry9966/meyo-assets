---
title: OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话
feedId: 35226
source: 综合讨论
publishedAt: 2026-08-29
---

## 背景

在 OpenClaw 里，主 Agent 调用子 Agent 处理长任务、多步搜索或代码生成，是很常见的自动化模式。但每次子 Agent 跑起来，中间推理、工具调用、临时结果都默认追加到主会话消息列表里。很快主上下文就会膨胀，主 Agent 开始把子 Agent 的报错当成用户输入，或者重复执行子 Agent 已经做过的步骤。问题不在子 Agent 能力不行，而在主会话没有边界。

## 问题

具体表现有几类：

- **上下文污染**：子 Agent 的每一步 `tool_result`、中间草稿都写回主会话，主 Agent 视野里全是噪音，token 消耗翻倍。
- **状态串味**：子 Agent 读写主 memory、环境变量或工作目录，返回后主 Agent 后续操作依赖的状态已经被改掉。
- **消息角色错乱**：子 Agent 的中间消息被标记成 `role:user` 回到主会话，主 Agent 误以为用户下达了新指令。
- **工具会话复用**：MCP 工具连接被多个子 Agent 共用，比如文件系统 MCP 的当前目录被子 Agent 切走，主 Agent 再操作时路径已经不对。

## 做法 / 步骤

我的做法是：把子 Agent 的 session 隔离到“只交结果，不交过程”。

**1. 定义返回契约**

主会话只接收子 Agent 的最终结果，不接收中间推导。返回结构建议固定为：

```json
{
  "ok": true,
  "summary": "任务完成情况",
  "artifacts": ["文件路径或关键输出"],
  "error": null
}
```

这样主 Agent 只看到一条 tool_call 和一条精简的 tool_result。

**2. 每次调用创建独立 session**

不要复用主会话的 session id。每次启动子 Agent 时生成新的 UUID 作为 session id，并设置回写模式为 summary。主会话只保留子 Agent 调用的元数据，不保留完整消息链。

示例思路：

```text
subagent.session_id = uuid()
subagent.parent_session = main_session_id
subagent.return_mode = "summary"
subagent.max_result_chars = 2000
```

具体字段名可能因 OpenClaw 版本而异，但核心是控制回写内容。

**3. 主会话侧做消息过滤**

如果框架支持 `session_policy`，可以设置：

```text
drop source=subagent and type != final
```

也就是只放行子 Agent 的最终结果，丢弃中间推理和工具过程。如果不支持，就在主 Agent 的接收器里做同样过滤。

**4. 状态隔离**

给子 Agent 分配独立的 memory scope，例如 `scope: "sub:<session_id>"`，或者使用独立临时目录。子 Agent 需要主会话数据时，通过参数显式传入，而不是去读全局 memory 或环境变量。

**5. 工具层隔离**

对 MCP 工具，给子 Agent 创建独立的 client/session，或者使用无状态工具。如果工具必须共享，确保子 Agent 完成后重置状态，比如恢复工作目录、清空临时变量。

**6. 异常兜底**

子 Agent 失败时，只返回简短错误码和日志文件路径，不要堆栈写回主会话。主 Agent 拿到错误码后可以决定重试或降级，而不是被一堆堆栈信息干扰判断。

## 踩坑点

- **以为关掉 verbose 就行**：verbose 只是日志输出，`tool_result` 默认全量回传，仍然会膨胀。
- **并行子 Agent 共用 session id**：多个子 Agent 同时跑，消息顺序错乱，主 Agent 分不清哪个结果对应哪个任务。必须每个子 Agent 独立 session id。
- **把中间消息标记为 user**：子 Agent 的思考过程一旦以 `role:user` 回到主会话，主 Agent 会认为用户有新指令，行为跑偏。
- **MCP 全局状态被改**：文件系统 MCP 的工作目录、环境变量、认证状态，子 Agent 一动，主 Agent 后续操作全乱。需要为子 Agent 独立 MCP session，或者结束时恢复现场。
- **日志直接打到主终端**：子 Agent 的 debug 输出混进主会话终端，影响输出解析。重定向到文件是最省事的。

## 可复用建议

- 封装一个 `spawn_isolated_subagent` 工具，内部处理 session 创建、结果裁剪、异常转换。主 Agent 只看到干净的结构化返回，不用关心隔离细节。
- 在主 Agent 的系统提示里明确：子 Agent 结果只作为参考数据，不把其中的内容当作新指令执行。
- 为子 Agent 设置独立工作目录和临时文件，避免文件系统冲突。
- 用完即弃：子 session 完成后归档或删除，不要长期保留。
- 如果子 Agent 需要返回大量中间产物，写文件并返回文件路径，而不是把内容塞回主会话。

## 总结

Session 隔离不是简单的开关，而是消息边界、状态存储、工具调用、异常处理四个层面的约束。核心是控制主会话看到什么：只让它看到子 Agent 的最终结论和必要产物，不让它看到推理过程和工具噪音。这样才能让子 Agent 真正并行、可替换，同时主会话保持干净。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/fecb00ce7db6587c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/698b6ab8befbb31b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/a137af4dc66d5460.png)

