---
title: OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话
feedId: 34583
source: 综合讨论
publishedAt: 2026-08-25
---

## 背景

在 OpenClaw 里用子 Agent 做插件调用、MCP 编排、自动化任务时，最容易出现一个问题：主会话被“淹没”。主 Agent 本来只需要拿到结论，但子 Agent 的多次工具调用、报错重试、中间 JSON、调试输出全写回了主会话。

表现通常有三类：

- 上下文窗口快速膨胀，主 Agent 开始丢早期关键信息；
- 主 Agent 被子任务的中间细节带偏，决策质量下降；
- 多个子任务并发时，历史记录交叉，排障非常困难。

## 问题

很多同学的默认做法是：把子 Agent 当成主会话内部的一个“函数调用”，让它直接继承主会话 transcript，或者把子 Agent 的全量输出 append 回主会话。

这相当于把子任务的草稿纸塞进主对话。主会话既是控制面，又是数据面。没有隔离时，子 Agent 的每一次工具调用、每一个中间结果，都会成为主 Agent 下一轮决策的噪声。

## 做法 / 步骤

我目前在 OpenClaw 里通常会这样隔离子 Agent：

### 1. 独立 session_id，主会话只存引用

子 Agent 启动时分配独立 `session_id`，主会话只保存 `child_session_id` 和任务状态，不持有子会话 transcript。

```json
{
  "session_id": "main-01",
  "child_session_id": "child-01",
  "parent_session": "main-01",
  "status": "running"
}
```

### 2. 父传子：最小上下文注入

不要主会话历史全量灌给子 Agent。只传任务输入、约束、关键环境变量。可以设置 `context_snapshot` 的最大 token 和最近轮次。

例如：

```json
{
  "task": "fetch_metrics",
  "args": { "host": "prod-01" },
  "context_snapshot": { "max_tokens": 800, "include_last_n": 2 }
}
```

### 3. 子回父：结构化返回协议

子 Agent 结束时返回结构化结果，主会话只写入摘要和引用，不回灌中间过程。

```json
{
  "status": "ok",
  "summary": "拉取到 3 个指标，其中 CPU 异常",
  "result_ref": "s3://bucket/artifacts/child-01/metrics.json",
  "artifacts": ["file_id:abc"]
}
```

### 4. 工具与 MCP 隔离

子 Agent 不要继承主会话的全部工具和 MCP 连接。使用工具白名单；MCP server 尽量 session-scoped 或独立实例，避免共享内存状态串味。

### 5. 生命周期管理

子任务结束主动关闭 session；设置 TTL 和最大步数，防止孤儿会话积压。

## 踩坑点

- **只存自然语言摘要会丢关键参数**：文件路径、资源 ID、环境变量、错误码必须结构化保存在 `result_ref` 或 `metadata`，不要指望主 Agent 从摘要里猜。
- **子 Agent 递归创建子 Agent 没有限制**：容易形成 session 树爆炸。设置 `max_depth` 并检查 `parent_session`。
- **并发子任务复用同一个 session_id**：状态会相互覆盖。每个子任务必须生成唯一 ID，并在日志里记录 `parent_session`。
- **过早清理子 session 导致 artifact 引用失效**：先落盘对象存储/文件，再更新主会话引用，最后关闭子 session。
- **MCP server 共享导致状态串扰**：缓存、连接池、临时表容易串。独立实例成本高，但至少要做命名空间或 key 前缀隔离。

## 可复用建议

- 封装一个 `ChildSessionManager`，统一负责创建、回收、TTL、返回协议校验。
- 子 Agent 返回必须通过 JSON Schema 校验，强制包含 `status`、`summary`、可选 `result_ref`。
- 主会话入口加 `summarize_for_parent` 过滤器：子 Agent 的中间日志不直接进主历史，只进审计日志。
- 给主 Agent 暴露子任务状态接口：`pending / running / ok / failed`，而不是暴露完整 trace。
- 关键任务保留 `child_session_id` 到审计日志，主会话只放引用，排障时按 ID 反查子 session 全量记录。

## 总结

session 隔离不是简单“开个子会话”就结束。核心是控制面与数据面分离：主会话拿结论和引用，子会话负责过程和工具调用。只有把上下文注入、返回协议、工具/MCP 作用域、生命周期都做成默认约束，子 Agent 才不会污染主会话。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/1645a3df7b7d3a02.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/2c5f9b344814fe7e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/ceea92d85a8cdb73.png)

