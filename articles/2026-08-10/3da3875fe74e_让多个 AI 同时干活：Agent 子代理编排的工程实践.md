---
title: 让多个 AI 同时干活：Agent 子代理编排的工程实践
feedId: 32414
source: 综合讨论
publishedAt: 2026-08-10
---

## 背景：为什么需要子代理并行

单 Agent 处理复杂任务时，常面临两个瓶颈：一是上下文窗口有限，强行把所有细节塞进一个对话容易丢失焦点；二是串行执行慢，比如要先查文档、再读代码、再生成报告，每一步都在等上一步完成。在 OpenClaw 这类 Agent 框架中，虽然可以通过工具调用逐步完成，但面对明显可以切分的任务（如同时起草多个模块文档、并行扫描多段代码的不同隐患），串行模式浪费了大量时间。

把任务拆成多个相对独立的子任务，交给专门的“子代理”并行执行，是提升效率的直观解法。子代理可以是独立的 LLM 调用、独立的 Agent 实例，或是通过 MCP 服务暴露的工具 Worker。它们各自拿着分好的活儿干活，最终把结果汇总。

## 问题拆解：并行编排要解决什么

编排多个子代理并行干活，不是简单把 prompt 分出去就算完。需要回答几个工程问题：

1. **任务怎么切**：主 Agent 如何将复杂任务分解成独立、可并行的子任务？
2. **接口如何定义**：子代理拿什么输入、返回什么输出，才能保证后续能可靠汇总？
3. **怎么做并行调用与结果收集**：用异步还是多线程？怎么等所有子任务结束或处理超时？
4. **出错怎么办**：某个子代理失败后是重试、降级还是中断整体流程？
5. **上下文隔离**：子代理之间是否需要交换信息？如何避免上下文污染？

在 OpenClaw 环境中，通常是通过 Tool 或 MCP 资源来管理子代理的启动和通信。下面给出一套可落地的实践步骤。

## 实践步骤：基于 asyncio + Tool 的并行子代理

假设我们要对一个代码仓库同时做三件事：**生成 API 文档**、**发现潜在性能问题**、**检查安全漏洞**。这三个任务互不依赖，完全可以并行。主 Agent 负责拆任务和汇总。

### 1. 定义子任务合约

用 Pydantic 定义每个子任务的输入和输出，保证数据结构稳定。

```python
from pydantic import BaseModel

class SubTaskInput(BaseModel):
    task_type: str  # "docs", "perf", "security"
    code_snippet: str
    file_path: str

class SubTaskOutput(BaseModel):
    task_type: str
    findings: str
    confidence: float
    error: str = ""
```

### 2. 实现子代理调用函数

每个子代理函数内部通过 OpenAI 兼容接口（或 OpenClaw 提供的 LLM client）调用模型，专门针对一种子任务。这里让 OpenClaw 的 `use_llm` 工具或直接使用 `OpenAI` 异步客户端。

```python
import openai

async def run_subtask(task: SubTaskInput) -> SubTaskOutput:
    system_prompts = {
        "docs": "你是一个技术文档专家...",
        "perf": "你是一个性能分析专家...",
        "security": "你是一个安全审计专家...",
    }
    messages = [
        {"role": "system", "content": system_prompts[task.task_type]},
        {"role": "user", "content": f"文件: {task.file_path}\n代码:\n{task.code_snippet}"},
    ]
    try:
        resp = await openai.ChatCompletion.acreate(
            model="gpt-4o-mini", messages=messages, temperature=0.2
        )
        return SubTaskOutput(
            task_type=task.task_type,
            findings=resp.choices[0].message.content,
            confidence=0.9,
        )
    except Exception as e:
        return SubTaskOutput(task_type=task.task_type, findings="", confidence=0, error=str(e))
```

### 3. 主 Agent 内编排并并行调度

在 OpenClaw Agent 的逻辑中（通常在某个 Tool 的实现里）使用 `asyncio.gather` 并发调度这些子代理。

```python
import asyncio

async def orchestrate(file_path: str, code: str):
    tasks = [
        SubTaskInput(task_type="docs", code_snippet=code, file_path=file_path),
        SubTaskInput(task_type="perf", code_snippet=code, file_path=file_path),
        SubTaskInput(task_type="security", code_snippet=code, file_path=file_path),
    ]
    # 并行执行
    results: list[SubTaskOutput] = await asyncio.gather(
        *(run_subtask(t) for t in tasks), return_exceptions=True
    )
    # 汇总并生成最终报告
    report = aggregate(results)
    return report
```

`asyncio.gather` 配合 `return_exceptions=True` 可以避免一个子任务失败导致整体中断，我们可以根据返回对象的 `error` 字段判断是否降级。

### 4. 汇总与校验

汇总函数对结果做健康检查：丢弃明显空或信心过低的结果，整合成 Markdown 报告。这里还可以再次调用一个总结用 LLM，将三条发现压缩成统一的交付物。

## 踩坑点

- **API 速率限制**：并发三个调用可能会触发 OpenAI 等服务的速率限制。需要实现指数退避重试，或限制并发数（用 `asyncio.Semaphore`）。
- **上下文膨胀**：每个子代理如果都传大量代码，输入 token 消耗是线性的，注意截断或提前提取关键片段。
- **错误传播**：`asyncio.gather(return_exceptions=True)` 让我们能拿到异常对象，但需要仔细判断是网络抖动还是模型持续返回乱码。建议给每个子代理设定 `asyncio.wait_for` 超时，并记录完整日志便于复现。
- **模型幻觉**：子代理产出的结构化回答可能偏离约定格式，要在汇总前加一层校验（如让主 Agent 用正则或小模型二次确认），避免把幻觉写进最终报告。
- **资源开销**：并行虽快，但 token 消耗、调用次数翻倍。建议只在任务明显可拆分且时效要求高时使用，日常轻量任务反而可能得不偿失。

## 可复用建议

1. **统一合同**：所有子代理遵循相同的 I/O 数据结构，方便流水线化。
2. **超时与重试**：为每个子任务设定 `timeout`，失败后可选重试 1～2 次，但控制总耗时。
3. **隔离上下文**：不要在主 Agent 的消息历史里混入子代理的中间输出，只保留最终结构化结果，避免污染后续推理。
4. **可观测性**：为每个子代理记录开始时间、结束时间、token 用量和结果摘要，方便分析瓶颈和成本。
5. **部分失败降级**：汇总步骤要能容错，例如三个子代理中有一个失败，依然可以产出包含两个维度分析的报告，并在报告中标注缺失项。
6. **考虑用 MCP 做 Worker**：如果子任务稳定且重用度高，可以写成 MCP server 提供远程调用，主 Agent 通过 MCP 客户端调用，有利于扩展和管理的解耦。

## 总结

子代理并行编排不是银弹，但面对分区明确的多样化分析任务时，能将分钟级的串行等待压缩到 20~30 秒内。核心在于**稳定的事先拆解、可靠的异常处理、结构化的结果校验**。在 OpenClaw 这类以工具和 MCP 为核心的 Agent 框架里，利用 asyncio 并发调度 + 明确的 I/O 合约，就能让多个 AI 真正“一起上”，而不是排队等咖啡。

---

