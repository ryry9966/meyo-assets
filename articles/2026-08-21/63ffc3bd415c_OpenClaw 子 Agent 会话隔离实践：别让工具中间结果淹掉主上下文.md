---
title: OpenClaw 子 Agent 会话隔离实践：别让工具中间结果淹掉主上下文
feedId: 34037
source: 综合讨论
publishedAt: 2026-08-21
---

## 背景

在 OpenClaw 里把子 Agent 当“工具”用已经很常见：让子 Agent 去查库、跑 MCP 插件、处理一段长文档、或者完成一次多步自动化。主 Agent 负责编排，子 Agent 负责执行。

但实际跑起来后，很多人会发现主会话越来越“脏”：子 Agent 的中间推理、MCP 返回的大段 JSON、工具报错栈、甚至子 Agent 从网页里读到的不可信文本，都跑进了主 session。结果是上下文快速膨胀，主 Agent 开始被无关信息干扰，甚至被恶意网页里的诱导指令带偏。

这个问题的本质不是“子 Agent 不该输出”，而是 **session 边界没有被明确设计**。

## 问题表现

典型的污染路径是这样的：

1. 主 Agent 调用子 Agent 处理一个任务。
2. 子 Agent 内部做了 3 次工具调用，每次工具输出都很长。
3. OpenClaw 默认把子 Agent 的完整事件流合并回主 session。
4. 主 Agent 看到的不只是一个结果，而是子 Agent 的“思考过程 + 工具中间态 + 最终答案”。

这会导致几个具体问题：

- **Token 浪费**：MCP 返回的长 JSON 可能占掉几万 token。
- **决策干扰**：主 Agent 读到子 Agent 的失败重试、工具报错后，容易做出过度保守或错误的判断。
- **安全风险**：子 Agent 读取的外部内容可能包含 prompt injection，如果原样带回主 session，就会污染主 Agent 的下一步行为。
- **调试困难**：主 session 里混合了不同层级的上下文，排查时很难定位是哪一步引入了脏数据。

## 做法与步骤

### 1. 明确子 Agent 的返回契约

子 Agent 不应该把内部过程“倒”给主会话。更合理的设计是：子 Agent 只返回一个结构化结果信封，例如：

```json
{
  "status": "ok",
  "summary": "仓库当前在 main 分支，工作区干净",
  "data": {
    "branch": "main",
    "dirty_files": 0
  },
  "errors": []
}
```

主 session 只需要这个信封，不需要子 Agent 的完整推理链和工具输出。

### 2. 给子 Agent 使用独立 session

在 OpenClaw 的子 Agent 配置中，显式声明隔离模式，并关闭上下文继承：

```yaml
subagent:
  session:
    mode: isolated
    inherit_context: false
    ttl: 600
  return:
    mode: final_only
    max_tokens: 2000
```

这里的关键是 `inherit_context: false` 和 `return.mode: final_only`。前者阻止子 Agent 自动继承主 session 的全部事件，后者保证只有最终消息会被回传。

### 3. 调用侧用 spawn，而不是 inline

如果直接在主流程里内联执行子 Agent，边界很容易模糊。建议用显式 spawn 调用：

```ts
const result = await openclaw.spawn({
  task: "check repo status",
  session: { isolated: true },
  return: "final"
});

await mainSession.post({
  role: "tool",
  content: JSON.stringify(result)
});
```

这样主 session 收到的只是 `result`，而不是子 Agent 的完整执行轨迹。

### 4. 对 MCP 工具输出做裁剪

很多污染来自 MCP 工具返回的超长输出。可以在子 Agent 的 MCP 客户端侧加一层过滤：

```yaml
mcp:
  output_filter:
    max_chars: 4000
    max_tokens: 1500
    include_fields: ["result", "status", "error"]
```

不要直接把原始 MCP 响应交给子 Agent，更不要让它原样回流到主 session。

### 5. 保留 sidecar trace 用于调试

隔离不等于丢弃过程。子 Agent 的执行日志可以写入独立的 sidecar session 或本地 trace 文件，方便排查，但不合并进主 session。

## 踩坑点

### 隔离不等于权限隔离

`session: isolated` 只隔离上下文，不隔离权限。子 Agent 仍可能继承主 Agent 的凭证、工具权限或环境变量。如果子 Agent 处理不可信内容，建议单独配置最小权限。

### final_only 仍可能带回脏数据

有些 Agent 框架里，子 Agent 的“final”消息里会包含它自己总结前的大段引用。比如：

```text
最终结论：任务完成。

以下是我参考的原始输出：
[ huge transcript ]
```

这等于把中间结果打包进了 final。需要在返回契约里明确禁止，或者用后处理把 final 消息截断到合理长度。

### 嵌套子 Agent 会放大污染

子 Agent 再 spawn 子子 Agent 时，如果每一层都往父级返回一点中间态，污染会逐层累积。建议在每一层都执行同样的 final_only 策略，不要只在最外层做。

### 调试时看不到过程

完全隔离后，主 session 很干净，但出了问题也不好查。解决方式是开启独立的 trace session，或者把子 Agent 关键步骤写成结构化日志，而不是塞回主会话。

## 可复用建议

1. **统一返回信封**：所有子 Agent 都返回 `status / summary / data / errors` 四字段。
2. **设置 token 预算**：子 Agent 的返回内容限制在 2000 token 以内，超出就截断或要求重写。
3. **MCP 输出必须有过滤器**：不要信任任何插件会返回“刚刚好”的内容。
4. **主 session 只接受结论型消息**：任何包含推理过程、工具原始输出、错误栈的消息，都不应该写入主 session。
5. **用 session graph 检查串流**：定期查看主 session 的事件来源，确认没有子 Agent 的执行事件被意外合并。

## 总结

OpenClaw 的子 Agent 会话隔离不是一个开关，而是一组边界设计：子 Agent 可以自由思考、自由调用工具，但最终只能把紧凑、可信的结论交回主会话。主 session 保留的是编排决策，不是执行现场。

做到这一点的核心只有三句：

- 子 Agent 用独立 session 跑。
- 中间态只写 trace，不写主 session。
- 返回值只允许结构化信封。

这样主会话才能保持轻量、可审计、不容易被工具输出或外部文本污染。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/5699d706d16ecd95.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/9be4433685ea5e8d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/ec060a44c1e2185c.png)

