---
title: OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话
feedId: 33561
source: 综合讨论
publishedAt: 2026-08-17
---

## 背景

在 OpenClaw 里接上 MCP、插件和自动化工具后，很多人会顺手把复杂任务拆给子 Agent：一个负责浏览器操作，一个负责代码执行，一个负责检索知识库。拆分本身没问题，但主会话会迅速变得不可控——上下文窗口被中间过程塞满，主 Agent 开始重复子任务、忽略原始指令，甚至把子 Agent 的日志当成用户输入来响应。

这不是模型能力问题，而是 session 污染。子 Agent 的过程数据、错误堆栈、工具返回 JSON、调试日志直接写回主会话，导致主会话既当指挥员又当回收站。正确做法是让子 Agent 在自己的 session 里跑，只回传结构化结果。

## 问题

默认配置下，子 Agent 继承主会话上下文，返回内容包括：

- 每一轮 reasoning 的完整文本
- MCP 工具调用的原始返回值
- 插件产生的长 JSON 或 HTML
- 错误堆栈和重试记录
- 子 Agent 内部的中间步骤

这些内容混入主会话后，会带来四个明显症状：

1. **指令稀释**：主 Agent 被大量过程文本包围，原始用户指令在上下文中的权重下降。
2. **重复执行**：主 Agent 看到子 Agent 未完成或报错，会尝试自己再跑一遍。
3. **token 爆炸**：一次子任务可能带来几万 token 的过程数据，很快触及上下文上限。
4. **提示词注入风险**：子 Agent 读取的网页、文件或 API 返回值如果未被过滤，可能直接改变主 Agent 行为。

## 做法/步骤

### 1. 给子 Agent 独立 session

不要用“在主会话里多写几轮消息”的方式模拟子任务。OpenClaw 里可以用独立的子 session 运行 Agent，父 session 只接收最终结果。

示例配置思路：

```yaml
sub_agent:
  session:
    mode: isolated
    parent_context: none
    return: final_only
  context_budget: 4000
  output:
    max_tokens: 800
    truncate: true
    strip_tool_logs: true
```

关键点是 `parent_context: none`，子 Agent 不自动继承主会话历史。需要传递的上下文，由父 Agent 显式打包成一段简短任务说明。

### 2. 只回传 final answer

子 Agent 运行过程中会产生多轮消息，但父会话只需要最终结果。可以把返回模式设置为 `final_only` 或 `last_message`，并限制返回内容。

如果子 Agent 返回的自然语言仍然过长，可以要求它返回固定 JSON：

```json
{
  "status": "ok",
  "summary": "登录页面已打开，未发现验证码",
  "artifacts": [],
  "next_action": "fill_credentials"
}
```

父 Agent 只读取 `summary` 和 `next_action`，其余字段按需裁剪。

### 3. 拦截工具/插件输出

MCP 工具返回的原始数据经常是污染源。比如一个搜索工具返回 5000 条结果，或浏览器插件返回整页 DOM。可以在子 Agent 侧对工具返回值做后处理：

- 白名单字段提取
- 限制数组长度
- 压缩 HTML 为纯文本摘要
- 错误信息只保留最后一行
- 二进制文件不进入上下文，只写磁盘并返回路径

如果 OpenClaw 的插件不支持输出过滤，可以在子 Agent 外再包一层工具适配器，统一处理返回值。

### 4. 分离日志与调试信息

调试日志、MCP 通信记录、堆栈信息应该写到文件或独立通道，不要拼进主会话的 message list。可以设置子 Agent 的 `log_destination: file`，主会话只感知任务状态。

### 5. 控制递归深度

子 Agent 再调用子 Agent，容易造成上下文指数级增长。建议限制 `max_depth: 1`，并规定子 Agent 只能调用白名单工具，不能再次 spawn 子 Agent。如果业务上需要多层，每层都必须做输出裁剪，且总预算不超过主会话的 20%。

## 踩坑点

- **只设了 temperature 或 max_tokens，没做 session 隔离**。这只能限制单次输出长度，过程污染依然存在。
- **子 Agent 返回“我分析了一下，然后……”**。如果没有结构约束，最终结果仍可能包含过程性描述。要强制返回 schema。
- **MCP 工具返回长列表时截断位置不对**。简单从头截断可能丢掉关键结果，应该先按相关性排序再截断。
- **手动裁剪历史消息时误删系统指令**。不要在父会话里硬删 message，应该从源头不让污染进入。
- **子 Agent 报错后，主 Agent 拿到空结果就重试**。建议子 Agent 出错时返回结构化错误对象，包含 `retryable` 和 `reason`，避免主 Agent 盲目重试。

## 可复用建议

把“隔离子 Agent”封装成一个固定模板，每次创建子任务时复用：

```yaml
spawn_isolated:
  context: minimal
  return: final_json
  tool_whitelist: []
  log_destination: file
  max_turns: 8
  max_tokens: 1000
  post_process:
    - strip_html
    - truncate_list
    - extract_summary
```

统一返回结构，能大幅减少主会话的解析负担。另外，每次运行前检查主会话 token 计数，如果子任务预算超过当前剩余窗口的 30%，就应该拒绝执行或压缩上下文，而不是硬塞进去。

## 总结

子 Agent 的 session 隔离不是“加个开关”就完事，需要从上下文继承、返回模式、工具输出、日志通道和递归深度五个方向同时收紧。核心原则只有一条：**父会话只接收决策所需的最小结构化信息，过程数据留在子 session 或文件里**。这样主 Agent 才能保持清醒，不被自己的工具链淹没。

---

