---
title: Subagent 并行编排实战：在 OpenClaw 中让多个 AI 同时干活
feedId: 31144
source: 综合讨论
publishedAt: 2026-08-01
---

## 为什么要让 subagent 并行工作

单 Agent 串行处理任务时，瓶颈非常明显。比如你要对一个代码仓库同时做安全审查、性能分析和可读性检查，如果逐个调用，总耗时就是三者之和。更合理的做法是把任务拆成独立部分，让多个子智能体并行执行，最后汇总结果。

OpenClaw 本身就支持构建复杂的 Agent 协作系统，结合异步调用和 MCP 工具封装，完全可以把“主 Agent 协调者 + 多个专业 subagent 并行运行”做成一个稳定、可复用的工程实践。这篇文章记录我在一个内部自动化流水线中的具体做法、遇到的问题和最后沉淀下来的复用结构。

---

## 问题拆解：并行编排需要的几个关键点

在做并行之前，有三个问题必须想清楚：

1. **任务拆分是否真的解耦**：如果 subagent 的输出相互依赖（比如 B 需要 A 的结果作为输入），强行并行只会增加协调复杂度，得不偿失。所以只对真正独立的子任务做并行。
2. **上下文如何隔离和传递**：每个 subagent 应该只拿到自己需要的上下文片段，而不是把整本“对话历史”全塞进去，否则 token 消耗会爆炸。
3. **错误和部分失败怎么处理**：并行意味着可能某个 subagent 超时、返回格式错误或干脆无响应，主 Agent 需要能优雅降级，而不是让整个调用链崩掉。

---

## 实践做法：基于 OpenClaw 的 Subagent 并行执行器

### 1. 定义专业 subagent

在 OpenClaw 中，每个 subagent 可以是一个独立的 `Agent` 实例，拥有自己的系统提示和工具集。例如，代码审查场景下可以定义三个子智能体：

- `security-reviewer`：专注安全漏洞检测，工具集包含代码扫描 MCP server。
- `perf-reviewer`：专注性能问题，可能连接 profiling 工具。
- `style-reviewer`：专注代码风格和规范，使用 lint 工具的 MCP 包装。

主 Agent 的作用是把原始输入（比如 PR diff）分发给它们，最后整合结果。

### 2. 异步并行调用

OpenClaw 的任务执行本身就支持协程。我们可以用 `asyncio.gather` 把对多个 subagent 的调用并行起来。关键伪代码如下：

```python
async def dispatch_subagents(coordinator, diff_context):
    subs = ["security", "performance", "style"]
    tasks = []
    for role in subs:
        sub_agent = get_sub_agent(role)   # 根据角色加载配置好的 Agent
        prompt = build_prompt(role, diff_context)
        tasks.append(coordinator.task(sub_agent, prompt))
    results = await asyncio.gather(*tasks, return_exceptions=True)
    return results
```

这里的 `coordinator.task(sub_agent, prompt)` 会触发一次到该 subagent 的完整调用，返回最终响应文本。`return_exceptions=True` 让单个子任务失败不会拖垮整个 gather。

### 3. 结果聚合与异常处理

拿到并行结果列表后，主 Agent 需要对每个 subagent 的输出做结构化解析（例如要求它们输出统一的 JSON 格式）。如果某个结果变成异常对象，就记录错误并生成降级说明，最后组装成汇总报告。

实际使用中，我会让每个 subagent 强制输出一个固定结构的 JSON：

```
{
  "status": "ok|fail",
  "findings": [...],
  "error": null
}
```

这样主 Agent 即使在没有 LLM 的情况下也能做基础合并，避免二次推理带来的幻觉和不稳定。

---

## 踩坑记录

### 坑 1：并行数过大触发 rate limit

如果所有 subagent 都调用同一个 LLM 后端，同时发起 5、6 个请求可能瞬间触发 API 速率限制。解决办法是限制并发数，可以用 `asyncio.Semaphore` 控制：

```python
sem = asyncio.Semaphore(3)
async def limited_task(sub, prompt):
    async with sem:
        return await coordinator.task(sub, prompt)
```

### 坑 2：上下文膨胀和成本翻倍

并行调用很容易把 diff 全文或长文档复制多份，导致 token 消耗是串行的 N 倍。实践中必须对输入做裁剪：安全审查 agent 只看增删代码行，性能 agent 只看热点函数上下文，风格 agent 只拿到改动文件路径和 lint 结果，而非完整 diff。

另一个技巧是利用 OpenClaw 的 memory 管理，每个 subagent 使用独立、短周期的 memory 空间，避免上下文污染和历史堆积。

### 坑 3：子任务间隐式依赖

有一次我把“检查拼写错误”和“检查注释是否完整”拆成两个 subagent，但实际上拼写检查结果会影响注释完整性判断，结果汇总时发现结论矛盾。这种隐式依赖需要重新审视拆分粒度，确保任务确实没有顺序关系。

### 坑 4：输出格式漂移

即使 prompt 里要求 JSON，有时候模型还是会返回多余的解释文字或 markdown 代码块包裹。稳健做法是在主 Agent 一侧做容错解析：尝试 strip 掉 markdown 标记、查找第一个 `{` 开始的子串，解析失败则记入异常并丢弃该 subagent 结果。

---

## 可复用建议：把并行模式沉淀为工具

经过几次迭代，我把上述逻辑抽象成一个可配置的 `ParallelSubagentRunner`：

- 接受一个主配置文件，定义 subagent 角色名、对应的 Agent ID、超时时间、最大并发数。
- 每个 subagent 对应一个 `task_template`，用占位符注入上下文片段。
- 内置 semaphore 流控和超时控制（`asyncio.wait_for`）。
- 输出合并策略可插拔：简单拼接、JSON 合并、多数表决等。

这样做的好处是，后续新增一个并行审查任务，只需加一份配置，不需要再碰协调代码。

同时，建议在 CI 流程中接入一个简单的“并行预算”监控：记录每次并行调用的总 token、耗时和成功率，当成本异常增长时及时调整并发策略或裁剪输入。

---

## 总结

subagent 并行编排并不是银弹，但在适合的场景下（独立的专业审查、多维度打分、多数据源交叉验证）能明显缩短端到端延迟。工程化的关键点在于：

- 只在真正解耦的任务上并行
- 严格控制每个 subagent 的上下文大小
- 用信号量、超时和异常捕获保证鲁棒性
- 用结构化输出 + 容错解析降低汇总风险

在 OpenClaw 生态里，这些能力都可以用现有框架灵活拼装出来，而且配合 MCP 工具化以后，subagent 不再只是一个 prompt 模板，而是带上工具、沙箱和资源约束的完整单元。并行编排的实践，也正是从“玩具 demo”走向“稳定生产”的一条必经路径。

---

