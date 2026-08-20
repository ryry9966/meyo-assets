---
title: Subagent 编排：让多个 AI 并行处理复杂任务的工程化实践
feedId: 33920
source: 综合讨论
publishedAt: 2026-08-20
---

## 背景

在 OpenClaw 的自动化实践中，单个 Agent 处理多步骤、多数据源任务时经常遇到三类问题：

- **上下文爆炸**：所有中间结果、工具输出、历史对话都堆在同一个上下文窗口里，导致模型注意力分散、幻觉增多。
- **串行等待**：每个步骤必须等上一步完成，长链路任务耗时线性增长。
- **失败难定位**：一旦中间出错，整个链路重跑，无法复用已完成的部分。

Subagent 编排把一个大任务拆成多个独立子任务，由主 Agent 负责分解、调度和汇总，多个子 Agent 并行执行。这种模式在信息收集、多文件分析、批量数据转换等场景非常适用。

## 问题

不是所有任务都适合并行。真正适合 subagent 的场景通常满足：

1. 子任务之间无强依赖，或依赖可以通过先并行后汇总解决；
2. 每个子任务的输入输出边界清晰，可以定义结构化输出；
3. 子任务失败不会影响其他子任务的执行。

如果任务本身是强顺序依赖（比如“先登录再抓取”），强行并行只会增加复杂度。因此，第一步是判断任务是否可拆分。

## 做法与步骤

以一个典型场景为例：**同时调研 5 个竞品官网并生成对比报告**。

### 1. 定义主 Agent 与 Subagent 角色

主 Agent 负责：
- 读取任务目标，生成子任务列表；
- 为每个子任务构造最小上下文；
- 并行触发 Subagent；
- 收集结果并校验；
- 汇总生成最终输出。

Subagent 只做一件事：接收一个明确指令，使用给定的工具集，返回符合 schema 的结果。

在 OpenClaw 中，每个 Subagent 可以配置独立的角色描述、MCP 工具集和输出格式。建议使用最小权限原则：只给它完成当前子任务必需的工具，避免误用。

### 2. 构造子任务输入

不要直接把整个对话历史传给 Subagent。只传必要字段，例如：

```json
{
  "id": "competitor_1",
  "website": "https://example.com",
  "focus": ["pricing", "features", "positioning"],
  "expected_output": {
    "pricing": "string",
    "features": "list",
    "positioning": "string"
  }
}
```

主 Agent 生成这样的结构化输入后，逐个调用 Subagent。

### 3. 并行触发与等待

在 OpenClaw 中，可以通过异步调用同时启动多个 Subagent。伪代码示意：

```python
subagent_tasks = [
    run_subagent(input_data=task)
    for task in subagent_inputs
]
results = await asyncio.gather(*subagent_tasks)
```

如果框架不支持原生异步，也可以用消息队列或任务池来实现。

### 4. 结果校验与重试

Subagent 返回后，主 Agent 用 JSON Schema 校验输出。不合格的结果触发一次重试（使用相同输入，可附加错误提示）。如果重试仍失败，标记为 `failed`，不阻塞其他任务。

### 5. 汇总输出

主 Agent 拿到所有子任务结果后，生成最终报告。汇总时只使用结构化数据，避免把原始工具输出再次喂给模型。

## 踩坑点

- **上下文污染**：Subagent 继承了主 Agent 的部分对话历史，导致输出跑偏。解决：显式构造独立 prompt，清空无关历史。
- **输出格式漂移**：模型不按 schema 输出，尤其是长文本或复杂 JSON。解决：在系统提示中给出输出示例，并用代码进行 schema 校验；校验失败自动重试一次。
- **并发写冲突**：多个 Subagent 同时写同一个文件或数据库表。解决：每个 Subagent 使用独立工作目录或输出键，主 Agent 负责合并。
- **超时与卡死**：个别 Subagent 可能因为工具报错或网络问题卡住。必须设置超时时间，超时后取消并记录错误。
- **成本失控**：并行数量过多导致 token 消耗成倍增加。设置最大并发数（例如 3～5），并优先执行高价值子任务。

## 可复用建议

1. **用配置文件定义 Subagent 模板**，不要每次手写 prompt。模板包括角色、工具集、输出 schema、超时时间。
2. **统一返回结构**：让每个 Subagent 返回 `{status, data, error}` 三字段，方便主 Agent 统一处理。
3. **记录每个 Subagent 的输入、输出、耗时**。日志是排障的唯一依据。
4. **失败隔离**：单个 Subagent 失败不影响整体流程，主 Agent 可以跳过或降级处理。
5. **封装 MCP 工具**：把外部 API 的认证、重试、限流逻辑封装在 MCP 工具层，Subagent 只调用标准化接口，减少重复出错。
6. **成本预算**：每次编排前预估 token 消耗，设置总预算和子任务数量上限。

## 总结

Subagent 编排的核心不是“让多个 AI 跑起来”，而是**任务拆分、边界清晰、失败隔离**。并行能降低总耗时，但会增加协调成本。只有子任务独立、输出可验证时，并行才划算。

在 OpenClaw 中落地时，从一个小场景（如 2～3 个并行子任务）开始，验证 schema 校验和日志链路，再逐步扩大规模。稳定比炫技重要。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-20/57f08f350103de67.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-20/5c6dd800f734ce6f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-20/d5e4095637d84960.png)

