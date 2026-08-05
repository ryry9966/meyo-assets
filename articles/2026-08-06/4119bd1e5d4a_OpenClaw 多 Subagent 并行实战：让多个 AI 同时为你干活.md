---
title: OpenClaw 多 Subagent 并行实战：让多个 AI 同时为你干活
feedId: 31784
source: 综合讨论
publishedAt: 2026-08-06
---

## 背景：单 Agent 的串行瓶颈

在 OpenClaw 生态里，我们习惯通过 MCP 工具扩展 Agent 的能力。一个典型场景：早晨到公司，想让 AI 一次性完成「代码检查 → 生成文档 → 编写测试」三步，结果发现 Agent 是老老实实地串行干活——先跑 linter，等 3 分钟；再读文件写文档，又 5 分钟；最后基于代码和文档出测试，再花 4 分钟。整个过程下来 12 分钟，其中大量时间花在等待工具返回、LLM 推理和 I/O 上。

这三个步骤之间其实没有强依赖：文档可以根据代码直接生成，测试也可以并行编写，最终只需要一次汇总。但单 Agent 的思考→行动循环天然是顺序的，即便你用异步调用，LLM 本身也只会一次给出一个 action。

**Subagent 并行编排**解决了这个问题。OpenClaw 支持将子任务拆解为多个独立的 subagent，让它们各自携带领域 prompt 与工具集，并行执行，最后汇总。这样原本 12 分钟的任务可以压缩到 4–5 分钟，效率直接乘三。

## 问题拆解

并行编排看似美好，但要落地必须面对几个工程问题：

1. **任务如何拆分**：以确保 subagent 之间尽量无依赖，或依赖可控。
2. **结果如何汇集**：上下文窗口会不会爆掉？格式不一致怎么办？
3. **资源冲突**：如果两个 subagent 同时写同一个文件，后果灾难。
4. **并发控制与容错**：LLM API 有频率限制，subagent 可能报错，要有退避重试。

接下来我会基于一个真实可复现的案例——「仓库日报自动化」——一步一步给出实践方案。

## 做法 / 步骤

### 1. 定义无状态 Subagent

在 OpenClaw 中，每个 subagent 是独立的配置项，通常放在 `subagents/` 目录下，YAML 格式。三个 subagent 分别负责：

- **code-reviewer**：扫描指定目录的代码，输出质量报告（JSON）。
- **changelogger**：分析最近的 git log，生成 CHANGELOG 条目（Markdown）。
- **doc-writer**：抽取函数签名和注释，更新 API 文档（Markdown）。

以 `subagents/code-reviewer.yaml` 为例：

```yaml
name: code-reviewer
model: claude-sonnet-4-20250514
system_prompt: |
  You are a code reviewer. Analyze the code under the given directory.
  Return a JSON object: {"issues": [{"file": "...", "severity": "high/medium/low", "message": "..."}]}
  Only use the provided MCP tools: file_reader, grep.
tools:
  - file_reader
  - grep
```

注意两点：**无状态**（不依赖外部会话），**结构化输出**（JSON 强制约束）。这保证了主 Agent 汇总时不需要再做复杂的自然语言理解。

### 2. 主 Agent 内的并行分发

在主 Agent 的 Action 配置中，使用 OpenClaw 内置的 `parallel` 指令或者通过 MCP 封装一个 `parallel_runner` 工具。这里我选择后者：编写一个轻量 MCP 服务（Python），暴露 `run_subagents` 工具，内部用 `asyncio.gather` 调用三个子任务。

核心逻辑示意：

```python
import asyncio
from openclaw_mcp import subagent_invoke

async def run_subagents(*requests) -> list:
    tasks = [subagent_invoke(req.name, req.input) for req in requests]
    results = await asyncio.gather(*tasks, return_exceptions=True)
    return [
        {"name": req.name, "result": r if not isinstance(r, Exception) else {"error": str(r)}}
        for req, r in zip(requests, results)
    ]
```

然后在主 Agent 的 system prompt 中允许它调用 `run_subagents`，并给出任务描述：

```
When asked to generate a daily report, call `run_subagents` with three tasks:
1. name: code-reviewer, input: {"directory": "/repo/src"}
2. name: changelogger, input: {"since": "1 day ago"}
3. name: doc-writer, input: {"source_dir": "/repo/src", "output_dir": "/tmp/docs"}
Then merge the results into a single daily report.
```

这样主 Agent 只需要一次工具调用，三个 subagent 就会并发执行。

### 3. 结果合并与文件写入

关键点：**不要让 subagent 直接写最终产物文件**，而是让它们返回结果字符串，由主 Agent 最后统一写入。这避免了文件锁和内容覆盖的问题。例如 doc-writer 只返回 Markdown 文本，主 Agent 收到后一次性调用 `write_file` 写入 `API-docs.md`。

对于超长的结果（如 code-reviewer 报了几十个 issue），我会在 subagent 的 prompt 里限制输出长度，比如“最多列出 20 个最严重问题”。这样上下文可控，汇总时不会超出模型窗口。

## 踩坑记录

### 1. 并发 API 限制

当 3 个 subagent 并发调用同一个 LLM 时，如果 API 的并发限制是 2，就会触发 429。解决：在 `asyncio.gather` 外包一层 asyncio Semaphore，控制最大并发数为 2。同时加上指数退避重试，处理偶发错误。

### 2. 工具调用冲突

虽然 subagent 各自有独立工具集，但如果它们共享了同一个 MCP 资源（比如同一个文件系统），加上某个粗心的 prompt 允许了写入，就可能在并行时产生竞态。实践教训：**只给 subagent 读权限**，写操作集中在主 Agent 完成。

### 3. 依赖悄悄混入

有一次我把代码审查的输出作为 doc-writer 的输入，破坏了并行性。后来明确：所有需要其他 subagent 输出的任务，都应该拆成两阶段——第一波并行完成独立分析，第二波再串行合并生成最终报告。第二波可以交给一个新的 subagent，或者直接由主 Agent 承担。

### 4. 上下文吞噬

汇总阶段如果直接拼接三个原始结果，token 数可能爆炸。对策：subagent 输出必须短小精悍，或者在主 Agent 的 prompt 里要求“用摘要格式合并”，利用模型自身的总结能力压缩。

## 可复用建议

- **契约优先**：每个 subagent 定义清晰的输入 JSON Schema 和输出 JSON Schema，不满足结构即视为失败，主 Agent 可据此触发重试或降级。
- **隔离工具集**：按最小权限原则给 subagent 分配工具，禁止它们直接修改仓库文件或调用外部写入 API。
- **并发控制常驻**：对所有外部 API 调用（LLM、文件系统、数据库）都加上 Semaphore 和 Retry，别心存侥幸。
- **用 OpenClaw 的内置能力**：避免自行实现 subagent 调度器，OpenClaw 的 MCP 框架已处理好凭证传递、日志追踪和错误传播，尽量站在巨人的肩膀上。
- **编排模式沉淀**：将常用的并行模式（比如分析-汇总、多视角审查）提炼为可重用的 MCP 工具或模板，团队成员只需配置 subagent 列表即可复用。

## 总结

在 OpenClaw 中实现多 subagent 并行，不是造一个分布式系统，而是利用现有框架做好任务拆分、并发控制与结果归并。当拆得干净、边界清晰时，三个 AI 同时干活带来的效率提升非常实在——原本需要十几分钟的日常任务，现在两三分钟就能出完整日报。对于任何希望将 AI 自动化深入工程流程的个人或团队，这套实践都是低门槛、高回报的基础设施。

下一步可以尝试引入更强的工作流引擎（基于 DAG 的有依赖任务编排），但那是另一个话题了。现在，动手把那些互不相干的重复劳动交给 subagent 们, 自己腾出时间去喝杯咖啡吧。

---

