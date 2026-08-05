---
title: Agent 的 subagent 编排：多个 AI 并行做事的实践
feedId: 31746
source: 综合讨论
publishedAt: 2026-08-05
---

## 背景

在 OpenClaw 里用 Agent 执行自动化任务时，单一推理链路很容易碰到“一步慢、步步慢”的瓶颈。典型场景如下：

- 需要同时查询多个外部 API（如新闻、研报、社交媒体）再汇总分析；
- 要对一批文档并行做摘要、实体提取或风险判断；
- 任务天然可拆分为独立子任务，但顺序执行耗时过长。

这些场景如果靠单个 Agent 循环调用工具，不仅慢，还容易出现上下文干扰和长对话漂移。更好的方式是把任务拆给多个 **subagent**，设计一个编排器（orchestrator）让它们并行工作，最后归并结果。

## 问题

**并行不是简单多开几个线程**。要落地的难点集中在三块：

1. **编排与上下文隔离**：每个子 Agent 只看到自己需要的上下文，避免无关信息污染和 token 浪费。
2. **并发与限速**：共享 LLM 后端时，会撞上 API 的 RPM/TPM 限制（特别是 OpenAI 的 429），没设计好就会全面拥塞。
3. **容错与部分失败**：一个子任务超时或返回格式错误，不能拖垮整个编排；还要能基于部分结果进行降级推理。

这篇文章基于 OpenClaw 的插件体系、Tool 抽象与 MCP 协议，给出一个可复现的工程化方案。

## 做法 / 步骤

### 1. 抽象子代理为可并行调用的工具

在 OpenClaw 中，可以把 subagent 封装成一个 `AsyncTool`，内部复用相同的 LLM backend，但限定系统提示词和允许的工具集合。例如一个多源信息收集场景，可以定义：

- `NewsSearcherSubagent`
- `PaperSearcherSubagent`
- `SocialSearcherSubagent`

每个子代理的输入输出用 Pydantic 模型约束，强制返回 JSON。

```python
class SearchResult(BaseModel):
    source: str
    summary: str
    url: str

class SubAgentInput(BaseModel):
    query: str
```

子代理内部调用 `agent.run()` 并解析输出为 `SearchResult`。

### 2. 搭建编排器

主 Agent（Orchestrator）接收用户任务，拆解为若干个子任务，然后并行调用这些 subagent tool。可以利用 OpenClaw 的 `call_tools` 并发执行，或直接用 `asyncio.gather`。

```python
tasks = [news_search(query), paper_search(query), social_search(query)]
results = await asyncio.gather(*tasks, return_exceptions=True)
```

`return_exceptions=True` 保证单个子任务抛异常不会中断整体。

### 3. 利用 MCP 传递共享上下文

如果多个子代理需要访问同一份文档或知识库，不要在每个请求里带全量内容。可以用 MCP 的 `resource` 暴露一个只读对象，子代理按需通过 `access_mcp_resource` 拉取，避免上下文膨胀。

### 4. 结果归并

编排器收集所有子代理返回的结构化结果，喂给一个“汇总 agent”进行二次推理，生成最终答案。即使部分子代理失败，汇总 agent 也能基于成功的部分给出有信息量的回复，并如实告知缺失的部分。

### 5. 超时与重试

为每个子任务单独设置 timeout，超时即取消。失败任务可以配置重试，但必须有最大重试次数和指数退避。OpenClaw 的工具执行层可以挂载全局的 timeout 中间件，减少样板代码。

## 踩坑点

1. **LLM provider 限速是头号杀手**  
   并行跑 5 个 subagent 时，几乎一定会触发 429。必须在编排器上加一个**令牌桶限速器**，根据 provider 的 QPS 调整并发数。OpenClaw 社区已有插件提供简单的 RateLimiter，可以按 RPM/TPM 配置。

2. **子代理输出结构不稳定**  
   即便 system prompt 要求返回 JSON，模型有时会夹带解释文字。解法是强制使用 structured output（如果模型支持），或者在工具层做后处理：用正则提取 JSON，失败了就用 fallback 的值。

3. **上下文悄悄膨胀**  
   如果不注意，每个子代理会拿到完整的对话历史。建议在调用前裁剪消息列表，只保留系统提示和当次任务指令，或者使用 summary 工具将长文本压缩。

4. **部分结果依赖导致“假并行”**  
   某些任务表面可拆，实际存在依赖（例如必须先拿到新闻再根据新闻去检索论文）。这种情形需要预先画 DAG，按层级并行，不能用 flat fan-out。

5. **资源泄露与孤儿任务**  
   编排器退出时必须主动取消所有仍在运行的子任务，否则会耗尽线程池或 HTTP 连接池。用 `asyncio.wait_for` 和 `TaskGroup` 配合清理。

## 可复用建议

- **统一的子代理基类**：封装超时、重试、结构化输出解析、日志，减少重复代码。
- **编排定义与执行分离**：用 YAML/JSON 描述子代理依赖和并行策略，运行时解析并构建执行图，方便调整。
- **本地 mock 测试**：使用 fake LLM backend 模拟不同返回格式和异常情况，提前验证容错逻辑。
- **成本监控**：每个子代理的 token 消耗单独打点，设置单次编排的总 token 预算，超标时触发止损。
- **善用事件总线**：OpenClaw 的事件系统可以记录每个子代理的 invoke、retry、error 事件，方便事后排障和优化编排图。

## 总结

并行 subagent 编排能显著提升 Agent 系统的吞吐与实时性，但它不是“免费午餐”。核心在于：**合理的任务拆分、严格的结构化合约、健壮的限速与容错机制**。在 OpenClaw 生态里，借助 Tool 抽象、MCP 上下文分发以及社区的限速插件，可以较干净地落地这套模式。希望这份工程实践能帮你在自己的自动化流里少踩一些坑。

---

