---
title: 主控调度与 subagent 并行：多 AI 干活的工程化拆分
feedId: 35074
source: 综合讨论
publishedAt: 2026-08-28
---

# 主控调度与 subagent 并行：多 AI 干活的工程化拆分

## 背景：为什么单个 Agent 不够用

在 OpenClaw 里跑复杂任务时，常见的问题是：一个主 agent 既要理解需求、调用 MCP 工具、又要写文件、做校验、处理异常。长上下文和串行工具调用会让任务越跑越慢，稍有一个环节失败就整体重来。尤其是那些本身没有依赖的步骤，比如“同时抓取三个数据源”“并行检查多个服务的健康状态”“对多个仓库做同一类修改”，串行执行纯粹是浪费时间。

于是我们开始用 subagent：主 agent 只做规划与调度，把独立子任务拆给多个 worker agent 并行执行，最后汇总结果。这并不高深，但要在工程上跑稳，有些细节值得写下来。

## 问题：并行不是简单开多个对话

一开始我在 prompt 里写“请并行执行以下任务”，结果并不理想。OpenClaw 的主 agent 虽然能调用 subagent 工具，但如果没有定义清楚子任务的边界，worker 之间会相互干扰：有的 worker 自己又去调用主 agent 的 MCP 工具，有的返回结果格式五花八门，还有的因为共享了同一个工作目录导致文件覆盖。更麻烦的是，一旦某个 worker 失败，主 agent 会卡在“等待结果”上，或者拿不完整的结果继续往下跑。

所以真正要解决的是三件事：

1. **子任务无状态化**：每个 subagent 只做一件事，输入输出明确。
2. **主控只做编排**：主 agent 不参与具体执行，只负责拆分、派发、合并、重试。
3. **结果结构化**：worker 返回 JSON 或固定 schema，方便主 agent 校验。

## 做法/步骤：一个可复现的并行编排

下面是我在 OpenClaw 中常用的结构，假设我们要并行检查三个 API 服务并生成报告。

### 1. 定义子任务 schema

先给每个 worker 一个明确的任务描述和输出格式。主 agent 的 prompt 中写入类似：

```text
你是主控 agent。你需要并行派发 3 个 subagent 任务：
- task_a: 检查 http://api-a/health，返回 {"service":"a","status":"ok|fail","latency_ms":number}
- task_b: 检查 http://api-b/health，返回同上
- task_c: 检查 http://api-c/health，返回同上

规则：
- 每个任务只返回 JSON，不要包含多余解释。
- 如果某个任务失败，不要阻塞其他任务，记录失败原因。
- 全部完成后，汇总成 {"results":[...], "summary":"..."} 输出。
```

### 2. 用 subagent 工具派发

在 OpenClaw 中，主 agent 可以通过 subagent 工具或插件来创建 worker。一般我会在工具定义里限制 worker 的能力：只允许网络请求和只读文件操作，不允许写共享目录。这样避免交叉污染。

```yaml
subagent_config:
  worker_profile: read_only_network
  max_parallel: 3
  timeout_seconds: 60
  retry: 1
```

`max_parallel` 控制并发数，避免外部 API 被瞬时打爆。`timeout` 一定要设，否则某个 worker 挂起会拖垮整个编排。

### 3. 主控合并结果

主 agent 收到所有 worker 返回后，先校验 JSON schema，过滤掉不合规的结果，再生成最终报告。如果某个 worker 失败，主控根据预定义策略决定是否重试或跳过。

## 踩坑点

### 1. 共享上下文和工作目录

多个 subagent 如果共用同一个工作目录，稍不注意就会互相覆盖文件。我的做法是：每个 worker 分配独立临时目录，或者干脆让 worker 无状态运行，只返回数据，不落盘。需要落盘的场景，由主控在最后统一写入。

### 2. 结果 schema 漂移

worker 有时会“自由发挥”，返回自然语言或字段不一致的 JSON。主控必须做强校验，最好在 prompt 里写死示例，并在代码层用 JSON Schema 验证。OpenClaw 里可以用 response_format 或自定义校验工具。

### 3. 并发限流与重试风暴

外部 API 有速率限制时，3 个 worker 同时请求可能触发 429，然后主控自动重试又加重负担。建议在主控层做限流：要么限制 max_parallel，要么加随机退避。我一般设 max_parallel=2，重试间隔 3-5 秒。

### 4. 错误隔离

一个 worker 失败不应影响其他 worker。主控要捕获异常，不要把失败的 worker 输出直接混入最终结果。可以用状态字段标记：`"status":"failed","error":"timeout"`，汇总时单独列出。

## 可复用建议

- **无状态 worker 优先**：子任务只依赖输入参数，不依赖前序结果。这样天然适合并行。
- **主控薄、worker 厚**：主控只做拆分和汇总，具体逻辑放在 worker prompt 或工具配置里。不要让主控既当裁判又当运动员。
- **结构化输出是底线**：JSON + schema 校验，不要指望 worker 返回“差不多”的自然语言。
- **设置硬超时**：每个 subagent 必须有 timeout，否则一个慢任务会卡住整个流程。
- **先小规模验证**：先用 2 个 worker 跑通流程，再扩大并发数。不要一上来就 10 个并行。
- **日志与追踪**：给每个 worker 一个 task_id，日志里带上，方便排查到底哪个子任务出问题。

## 总结

Subagent 并行编排的价值不在于“多个 AI 同时说话”，而在于把复杂任务拆成边界清晰、可重试、可观测的小单元。主控负责编排，worker 负责执行，结果用结构化数据回流。这样即使某个子任务失败，整体流程依然可控。

在 OpenClaw 环境里，这套模式配合 MCP 工具和插件能力会更好用：外部能力通过 MCP 暴露给只读 worker，主控则用 subagent 工具做并行调度。跑稳之后，你会发现自己不再需要一个“万能 agent”，而是一组可组合、可替换的 worker。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/51c7aec98ff7c39e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/42ac8241fa372c03.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/c51ce38c67897e0a.png)

