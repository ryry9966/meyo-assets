---
title: OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话
feedId: 35617
source: 综合讨论
publishedAt: 2026-09-01
---

## 背景

在 OpenClaw 里把自动化任务拆给子 Agent 之后，最常遇到的不是子 Agent 能力不够，而是主会话被慢慢污染。子 Agent 的中间推理、工具调用返回、重试日志、插件输出会一起混进主上下文。主模型每轮都要重读这些内容，跑几步之后就开始注意力漂移。

## 问题

主 session 承担两件事：决策和短期记忆。子 Agent 如果直接继承父 session，等于把执行过程也写进了主上下文。表现通常是：

- token 消耗快速上升；
- 后续指令被某个无关报错带偏；
- 重跑同一任务时，主会话状态不确定，难以复现；
- MCP 工具状态被串用，子任务之间互相影响。

本质不是“信息太多”，而是信息粒度错了。主会话需要的是结论，不是完整 trace。

## 做法 / 步骤

1. **显式创建独立 session**  
   子 Agent 不要省略 `session_id`。让它使用独立 session，只把结果传回主会话。

2. **定义回传协议**  
   子 Agent 只返回结构化结果，例如：

   ```json
   {
     "outcome": "completed",
     "evidence": "已生成 3 个候选方案",
     "risks": "第 2 个方案需要人工确认参数"
   }
   ```

   不返回逐轮日志、不返回完整工具输出。

3. **主会话用工具调用方式读取结果**  
   主 session 读取子 Agent 的返回摘要，不订阅子 session 的事件流。

4. **设置截断与落盘**  
   类似这样的配置思路，按当前版本调整字段：

   ```yaml
   sub_agent:
     isolated_session: true
     reply_mode: structured_summary
     max_reply_tokens: 1200
     drop_intermediate: true
     trace_sink: file:///tmp/oa-trace/{trace_id}.jsonl
   ```

   原始 trace 写到外部文件，主 session 只保留 `trace_id`。

## 踩坑点

- **只截断不回摘要**  
  子 Agent 如果最后只返回一个 `done` 或空结果，主会话没有决策依据。需要显式生成 outcome 摘要。

- **隔离了 session，但没隔离 MCP 状态**  
  同一个数据库 cursor、文件句柄、浏览器实例仍然可能串。给子 Agent 新建 scoped client，结束时主动 close。

- **把子 Agent verbose 日志短期接回主 session**  
  调试时确实方便，但时间一长主会话就被错误信息污染。需要看日志时用 trace 文件查，不要回灌。

- **环境变量和插件配置泄漏**  
  主会话临时变量不要直接传给子 Agent。显式传参，用完清理，避免跨 session 留下共享状态。

## 可复用建议

- 主会话只保留三类信息：`goal`、`current_status`、`next_action`。
- 子 Agent 返回控制在 800–1500 token 内，原始过程全部落盘。
- 每个子任务都带 `trace_id`，排查问题只查 trace，不改主上下文。
- 给子 Agent 设置独立的 MCP 命名空间和 env scope，任务结束触发清理钩子。

## 总结

session 隔离不是拒绝信息，而是改变信息回传粒度。理想的子 Agent 应该像 API：请求明确、返回结构化、内部过程可观测但不进入主上下文。这样主会话才能保持轻量，多步自动化才能稳定跑下去。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/e72a7c952a9f7c46.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/4e982938077931bc.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/c52b2f2c32777838.png)

