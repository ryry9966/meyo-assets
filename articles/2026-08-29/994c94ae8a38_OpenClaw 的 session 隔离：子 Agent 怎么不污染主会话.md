---
title: OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话
feedId: 35167
source: 综合讨论
publishedAt: 2026-08-29
---

## 背景

在 OpenClaw 里，主会话是控制平面，子 Agent 负责具体执行：搜索、读文件、跑自动化、调用 MCP。一个很常见的事故是：子 Agent 返回了大段原始结果，主会话开始变慢、上下文爆炸，甚至主 Agent 把子 Agent 的中间推理当成自己的决策依据。

问题不在于子 Agent 能不能做，而在于它怎么把结果交回来。

## 问题拆解

主会话被污染，通常来自三条路径：

1. **直接回灌 transcript**：把子 Agent 的完整对话 append 到主 session。
2. **上下文继承**：子 Agent 带着主 session 历史启动，把主会话的 system prompt、权限声明、错误假设一起复制。
3. **无边界共享**：MCP 工具或文件系统状态被子 Agent 修改后，主 Agent 后续调用读到脏数据。

比如主 Agent 需要查 20 个页面，子 Agent 逐个抓取。如果每次抓取都返回全文，200k 上下文很快会被撑满，而且真正有用的结论被淹没。

## 做法

### 1. 子 Agent 独立 session，只注入任务上下文

给每个子 Agent 新建 session，显式记录 `parent_session_id`，但不要继承主历史。传入内容只保留：任务描述、输入参数、输出 schema、时效/资源限制。不要让主 session 的完整 messages 进入子 Agent。

### 2. 强制结构化返回，不返回原始 transcript

让子 Agent 只返回摘要和引用，不返回完整日志。例如：

```json
{
  "ok": true,
  "summary": "找到 3 个相关文档，其中 1 个与当前问题直接相关。",
  "artifacts": ["/workspace/task-123/search_result.md"],
  "error_code": null
}
```

完整内容写文件，主会话只保留路径。摘要控制在 300-500 tokens 内。

### 3. 主会话用 tool 结果承载子 Agent 回传

不要把子 Agent 的 assistant 消息直接加入主对话。建议在主会话里以 tool/context 角色写入：

```ts
const sub = await spawnSubagent({
  task,
  parentSession: mainSessionId,
  isolated: true,
  returnMode: 'summary',
  maxOutputTokens: 1200,
});

await appendToMainSession({
  role: 'tool',
  content: JSON.stringify({
    status: sub.ok ? 'ok' : 'error',
    summary: sub.summary,
    artifact: sub.artifactPath,
    subSessionId: sub.sessionId,
  }),
});
```

这样主 Agent 知道这是工具执行结果，不会把它当成自己的推理。

### 4. 给子 Agent 单独的 system prompt

子 Agent 的 system prompt 要明确：

- 不要复述主会话背景；
- 不要把中间推理写进 summary；
- 失败时返回 `error_code`，不要贴完整堆栈；
- 生成的文件放在独立目录，不要覆盖主任务文件。

### 5. 设置资源边界

给子 Agent 配 `max_steps`、`timeout`、`max_context_tokens`、`max_output_tokens`。子 Agent 上下文越小，它越不容易把无关内容带回。

## 踩坑点

- **回传网页全文**：子 Agent 搜到内容后贴 5000 字正文。解决：只回传 `title/url/3 行摘要`，全文存 artifact。
- **MCP 工具返回大 JSON**：自动回填后主上下文爆炸。解决：在 MCP wrapper 层过滤字段，比如只保留 `items[].title`、`items[].url`、`items[].summary`。
- **session id 串号**：嵌套子 Agent 时，子 session id 被误写进主 session，后续消息错乱。解决：主 session 只记录 `sub_session_id` 作为引用，不参与对话。
- **主会话 system prompt 被污染**：子 Agent 的“你是 XX 助手”如果回传，可能覆盖或混淆主 Agent。解决：子 Agent 返回内容禁止带角色声明。
- **孤儿 session 未清理**：短任务跑完留下大量 session。设置 TTL，定期清理。

## 可复用建议

- 封装 `spawn_isolated_subagent` 工具，主 Agent 只能看到 `{status, summary, artifact_path, session_id}`。
- 所有子 Agent 输出先走压缩器：摘要 + 关键引用 + 文件路径。
- 给子 Agent 单独配置只读权限或受限 workspace，减少污染源。
- 监控主 session token 构成，如果子 Agent 回传占比持续超过 20%，说明隔离没做好。
- 约定一个固定回传 schema，避免每个子 Agent 自己发明字段。

## 总结

Session 隔离的本质是让主会话保持“控制平面”，子 Agent 留在“数据平面”。主会话需要的不是所有细节，而是可验证的结论和可追踪的引用。做到这一点后，主 Agent 会更稳定，上下文窗口也不会被无意义撑大。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/9df28fde0b0e3c93.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/575076d6393b915f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/0a228b73f3b065f1.png)

