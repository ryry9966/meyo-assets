---
title: Subagent 编排实践：让多个 Agent 并行做脏活，主控只做验收
feedId: 34586
source: 综合讨论
publishedAt: 2026-08-25
---

## 背景

在 OpenClaw 这类 Agent 环境里，单 Agent 处理长任务时容易遇到三个问题：上下文窗口被中间结果撑爆；多步任务串行执行，慢且容易在步骤间漂移；主控既要拆解任务又要执行细节，错误恢复困难。后来我把执行层拆成 subagent，主控只做拆解、分派与验收。这篇文章记录可复用的编排方式。

## 问题

不是所有任务都适合并行 subagent。适合的场景通常有三个特征：

1. 子任务之间没有强顺序依赖；
2. 每个子任务只需要局部上下文；
3. 输出可以结构化或摘要化。

比如一次发布前检查：查日志、跑回归、读变更、核对配置，四件事可以并行。不适合的是“先改代码再跑测试再部署”这种流水线。

## 做法/步骤

### 1. 主控与子代理分离

主控 agent 不执行具体业务，只负责：

- 把任务拆成可独立验收的子任务；
- 给每个子代理一个明确上下文切片；
- 规定输出 schema；
- 合并结果并决定是否重试。

子代理则被当成一个普通 tool 调用，但内部是独立的 agent run。工具列表只给最小集合，例如 `read_file`、`grep`、`mcp__logs_read`。不要给子代理写权限，除非任务本身要求落盘。

一个配置思路：

```yaml
subagents:
  log_check:
    prompt: "只检查 gateway 最近 30 分钟错误日志，输出 JSON。"
    tools: [read_file, grep]
    output_schema:
      findings: list
      severity: string
  config_check:
    prompt: "对比 staging 与 prod 配置差异，只报告会影响启动的项。"
    tools: [read_file, diff]
    output_schema:
      diff_items: list
      risk: string
```

### 2. 并行调度

在主控里用并发原语同时触发多个 subagent。OpenClaw 的 tool call 如果支持 async，就 `await asyncio.gather(*calls)`；如果只支持同步，就在外层包线程池。并发数控制在 3-5。并发过高会导致 MCP server 限流或上下文调度抖动。

我通常设置 `max_concurrency=3`，一个任务最多重试一次。超时时间按子代理最长耗时乘 1.5 设置，避免一个子代理卡死拖垮整批。

### 3. 结果合并

主控收到各子代理输出后，不直接拼接。先做三件事：

- 校验 schema，解析失败就丢弃并标记 `parse_error`；
- 去重和冲突检测：同一文件被多个子代理报告时，取更具体的那个；
- 留下证据：保留子代理的 trace_id、耗时、token 消耗、退出原因。

这样主控的最终输出是一份“验收报告”，而不是一堆中间文本。

## 踩坑点

1. **子代理上下文太窄**：只给子代理看局部日志，它会编造不存在的文件路径或 MCP 结果。解决方法是允许子代理在输出中声明“未见证据”，并禁止它推断上下文外的事实。
2. **并发写冲突**：两个子代理同时写同一文件或同一数据库 key。建议子代理默认只读，写操作全部回收给主控串行执行。
3. **输出格式漂移**：子代理聊 high 了会输出一段散文而不是 JSON。可在 prompt 末尾加“只输出 JSON，不要解释”，并在解析层做正则兜底。
4. **MCP session 复用问题**：多个子代理共用同一个 MCP session 时，有些 server 不支持并发请求，会报 `session closed` 或返回交错结果。每个子代理建立独立 session，或对 MCP 调用加锁。
5. **无限嵌套**：子代理再调子代理，层级一深 token 消耗指数上升。限制最大深度为 1-2 层，超过就让子代理把任务退回主控重拆。

## 可复用建议

- 子代理 prompt 模板固定：`背景 + 任务 + 只读边界 + 输出 schema + 停止条件`。
- 给子代理传入快照版本号，避免执行期间文件被外部改动。
- 主控只做路由和验收，不替子代理写执行细节。
- 记录每次 subagent 调用的 trace_id、耗时、token、退出原因，便于事后排查。
- 并行度用 feature flag 控制，先跑串行基线，再逐步放开。

## 总结

Subagent 编排的价值不是“多个 AI 同时聊天”，而是把主控从执行细节中解放出来，用隔离上下文控制风险。实践中，把子代理当工具、限制只读、结构化输出、并发控制，这四点比换模型更能提升稳定性。如果任务本身有强顺序依赖，就不要硬并行，老实串行反而更省 token。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/4f65714a5eda40ef.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/5fd7ce8a6ed39659.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/87ea2e7b240c4112.png)

