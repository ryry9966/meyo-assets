---
title: Subagent 并行编排实战：在 OpenClaw 中用多 AI 同时处理密集任务
feedId: 32566
source: 综合讨论
publishedAt: 2026-08-11
---

## 背景：单 Agent 串行已经到了瓶颈

在 OpenClaw 里，单个 Agent 本身已经能打通 MCP 工具、浏览器、文件系统和外部 API，但面对“高密度多条目”任务时，串行执行会迅速暴露短板。典型场景：

- 批量抓取 20+ 个 RSS 源并生成摘要
- 同时审查一个 PR 中改动的 10 个文件
- 对一组用户问题并行调用不同 MCP Server 进行查询

如果让一个 Agent 逐条处理，即使每个任务只要 3 秒，30 条就要一分半钟，而且中间任意一个工具调用阻塞，整条流水线卡死。OpenClaw 从某个版本开始稳定支持 Subagent 机制，这为“一个主 Agent + 多个 Worker Agent 并行干活”提供了工程化基础。

本文不泛泛聊概念，直接给出可复用的并行编排方案，以及实际踩坑后的纠正策略。

## 问题定义：多个 Subagent 并发时的三个核心矛盾

1. **结果顺序性**：并发返回顺序不可预期，而调用方往往要求“第 i 个输入对应第 i 个输出”。
2. **资源与限流**：Token 消耗暴增、OpenAI/Claude API 的并发限制、MCP Server 连接数上限。
3. **部分失败**：一个子任务超时或返回异常时，如何不影响其他任务，又能统一收集错误。

一个朴素的 `asyncio.gather` 并不能直接解决这些矛盾，需要一层薄薄的调度封装。

## 做法：一个可复用的 ParallelAgentRunner

以下是基于 OpenClaw 的 Agent 抽象实现的并行调度器思路（避免贴大段代码，用关键设计说明）。

### 1. 定义 Worker Agent 模板

不在运行时动态创建复杂 Prompt，而是提前设计好通用的 Worker 指令模版，利用 OpenClaw 的变量注入机制将任务上下文传入。例如：

```text
你是一个 RSS 摘要 Worker。
你的任务是：读取 {{rss_url}}，提取最近 5 篇文章的标题和 50 字摘要，
并以 JSON 格式输出：{"url": "{{rss_url}}", "items": [...]}
只输出 JSON，不要多余解释。
```

注意：要求 Worker 只输出结构化数据，是为了方便主 Agent 收集解析。不要让它自由对话，否则后续合并成本极高。

### 2. 创建并行调度器类

关键要点：

- **保持输入顺序**：提前为每个任务生成一个 `task_id`（如索引），Worker 返回时将 `task_id` 一并带回。
- **并发控制**：使用 `asyncio.Semaphore` 限制同时运行的 Worker 数量（建议从 3 开始，根据 API 限流调整）。
- **超时与重试**：每个 Worker 设置 `timeout`（例如 45s），并引入指数退避重试（可用 `tenacity` 库），最多重试 2 次，避免长时间挂起。
- **隔离上下文**：每个 Subagent 调用时，不传递主 Agent 的完整历史，仅注入当前任务变量，防止上下文污染和 Token 浪费。

伪流程：

```text
for each task in tasks:
    acquire semaphore
    launch async run_subagent(task_id, task_input)
    store future with task_id

results = []
for task_id in input_order:
    await corresponding future
    if success: results.append(parsed_output)
    else: results.append(fallback_error_entry)
```

### 3. 主 Agent 只做编排和聚合

主 Agent 本身不执行具体的工具操作，而是：

- 拆分任务列表
- 启动并行 Worker
- 收集结构化结果
- 根据汇总结果进行最终决策或格式化输出

这样责任边界清晰，主 Agent 的上下文永远干净，不会因为并行滥用导致 Prompt 爆炸。

## 踩坑记录

### 坑 1：Subagent 返回格式不遵守约束

即使 Prompt 里强调“只输出 JSON”，某些模型仍会包裹解释文字。解决方案：在解析 Worker 输出时，用正则提取首个 JSON 对象（`re.search(r'\{.*\}', text, re.DOTALL)`），并在失败时触发重试，同时增加 `temperature=0.1` 降低发散。

### 坑 2：MCP 连接池耗尽

如果 10 个 Worker 同时访问同一个 MCP Server（如本地 SQLite 工具），可能触发连接数限制或锁冲突。解决：对每个 MCP Server 设置专用的 `asyncio.Lock`，同一时刻只允许一个 Worker 调用该资源，其他资源可以并行。或者直接在 Worker 模板里规避非必要的 MCP 调用，让 Worker 尽量“纯脑力”输出。

### 坑 3：速率限制并发杀手

OpenAI 的 RPM/TPM 限制在被并行调用急速消耗时，会返回 429 错误。简单的 `tenacity` 重试如果没有配合 `max_concurrency` 限制，反而会雪崩。建议在调度器层设置 `min_delay_between_calls`（例如 0.5 秒），降低瞬时冲击，并在遇到 429 时进行 10~30 秒退避。

## 可复用建议

1. **封装通用 Runner**：把并发控制、超时、重试、结果排序抽象为一个 `ParallelAgentRunner` 工具类，后续所有并行场景只需传入 Worker 模板和任务列表即可。
2. **Worker 模板化管理**：将常用 Worker 指令存放在独立的 YAML/Markdown 文件中，主 Agent 根据场景选择加载，避免每次都重新设计 Prompt。
3. **监控与降级**：在 Runner 中记录每个 Worker 的耗时、成功/失败状态，当总失败率超过 30% 时，自动降级为串行模式，保证至少拿到部分可用结果。
4. **使用 MCP 返回结构化数据**：如果 Worker 需要查询外部数据，让 MCP Tool 输出 JSON Schema 约束的结构，而不是自由文本，减少后续解析成本。
5. **逐步提升并行度**：先在测试环境用 2 并行度验证逻辑正确，再调至 5、8，观察 API 错误率和延迟，找到最佳吞吐点。

## 总结

Subagent 并行编排不是简单的“多开几个 Agent”，而是在顺序性、资源隔离和容错之间找到工程化平衡。通过模板化 Worker、可靠的调度器封装以及结构化输出约束，OpenClaw 可以将原本一维的串行处理升级为多维并发的数据处理管线，显著降低端到端延迟，同时保持不错的可维护性。

下一步可以尝试将 Runner 与 OpenClaw 的 Workflow/Memory 模块集成，形成“主 Agent 决策分支 + 并行 Subagent 集群执行”的稳定模式，这会让自动化 pipeline 的弹性大大增强。

---

