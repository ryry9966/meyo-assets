---
title: 构建对故障友好的 AI Agent：外部 API 失效时的恢复策略
feedId: 31336
source: 综合讨论
publishedAt: 2026-08-02
---

## 1. 背景：Agent 的命脉绑在外部服务上

在 OpenClaw 或基于 MCP 的 Agent 架构里，Tool 几乎一定是外部 API 的封装——查询天气、搜索知识库、调用企业内部接口。这些外部依赖的行为模型很简单：**不可靠是常态**。网络抖动、限流阈值被触发、认证 token 过期、下游服务部署导致短暂 503，都会让一次 Tool 调用失败。

如果我们把异常直接抛给 Agent 的推理循环，轻则单次任务中断，重则一个长工作流的前面步骤白费，用户面对“抱歉，我无法完成”只有无奈。要做出工程上实用的 Agent，错误恢复不能是附加项。

## 2. 问题拆解：为什么不能无脑重试

最直觉的反应是“失败就重试”，但会遇到一系列工程陷阱：

- **雪崩**：在短暂宕机时，大量 Agent 实例同时重试，瞬间压垮刚恢复的服务。
- **非幂等操作重试风险**：创建订单、发送通知这类操作，简单重试可能造成重复副作用。
- **错误类型不分级**：401/403 该立即失败，而不是重试；404 可能是永久缺失资源。
- **用户感知差**：等待 3 次重试耗光超时，用户早已离开。

所以我们需要一套分层分级的恢复方案，而不是一个 `while(retry_count < 3)`。

## 3. 做法与步骤：分层恢复设计

### 3.1 错误分类器

首先把捕获的异常映射成三个类别：

- **瞬时可恢复**（Retryable）：503、连接超时、短暂 DNS 解析失败。
- **明确不可恢复**（Fatal）：401、403、404、422 业务逻辑错误。
- **语义不明确**（Unknown）：未预期的 status code，先归类为可有限次重试，但要带上熔断。

代码层可以做一个简单的分类函数：

```python
from aiohttp import ClientError

def classify_error(exc: Exception, status_code: int | None) -> str:
    if status_code in (429, 503):
        return "retryable"
    if status_code in (401, 403, 404):
        return "fatal"
    if isinstance(exc, (ConnectionError, TimeoutError)):
        return "retryable"
    return "unknown"
```

### 3.2 重试策略（指数退避 + 抖动）

针对 `retryable` 和首次遇到的 `unknown`，采用 **指数退避 + 随机抖动**，上限设为 3 次、最大等待时间不超过 30 秒。这样做既避免盲目重试，也防止「惊群」：

```python
import asyncio
import random

async def with_retry(func, *args, retries=3, base_delay=1.0):
    for attempt in range(retries):
        try:
            return await func(*args)
        except Exception as e:
            status = getattr(e, "status", None)
            cat = classify_error(e, status)
            if cat == "fatal":
                raise
            if attempt == retries - 1:
                raise
            delay = base_delay * (2 ** attempt) + random.uniform(0, 1)
            await asyncio.sleep(delay)
```

### 3.3 断路器（防止持续无效请求）

按服务粒度维护断路器状态（Closed / Open / Half-Open）。一个简单的内存实现：连续失败 5 次进入 Open，阻断所有请求 60 秒；之后转为 Half-Open，放行一部分请求探路，成功则恢复正常，失败则继续断路。Python 可直接使用 `pybreaker` 库，也可基于 Redis 做跨进程熔断。

对于 Agent 而言，断路期间 Tool 调用直接触发预设的降级逻辑。

### 3.4 优雅降级（Fallback）

不是所有 Tool 都不可替代：

- 天气查询失败，返回最近一次缓存的天气，并附上 `"数据非实时，更新时间 X 分钟前"`。
- 知识库搜索失败，回退到 Agent 自身的内部知识（RAG 的本地索引）。
- 非关键步骤：跳过并记录。例如自动生成报告埋点失败，不能阻塞整个报告。

Fallback 响应要设计为 `ToolResponse`，让 OpenClaw 的规划层能够区分正常结果和降级结果，方便后续摘要给用户。

### 3.5 用户透明提示

任务完成后，在结果末尾明确告知降级部分：

> 「提醒：支付状态查询因服务暂时不可用，展示的是 10 分钟前的缓存结果，请手动确认。」

这样保持信任，也给人工核对留出空间。

## 4. 踩坑点

- **不区分幂等操作**：我曾看到一个自动采购 Agent，下单接口因网络错误重试 3 次，最终创建了 3 张相同的采购单。必须在调用前识别幂等性，非幂等操作需要去重 Key 或只重试明确读操作。
- **断路器半开状态一过立即压垮**：半开时没有做流量探针，直接放行大量积压请求，服务再次熔断。正确做法是半开时只允许 1-2 个请求探测。
- **Fallback 数据造成错误决策**：股票价格降级为 1 小时前的价格，Agent 据此做出了交易建议。降级数据要加上置信度和时间衰减，并在 Planner 层面判断是否可用。
- **错误日志不关联链路 ID**：无 trace id，很难追踪用户报障。建议每个 Agent 任务挂上唯一的 run_id，贯穿所有 Tool 调用和重试。

## 5. 可复用建议

- **封装 ResilienceToolWrapper**：在 MCP Tool 的上层做 AOP 横切，统一处理重试、断路、降级，让业务开发者只需配置策略 YAML。
- **配置驱动**：为每个 Tool 设定 `retry_config`、`circuit_breaker_settings`、`fallback_handler`，集中管理。
- **结构化错误返回**：不让 Agent 直接面对 Python traceback，统一返回 `ToolResult(error_code="UPSTREAM_UNAVAILABLE", retryable=True, fallback_data=...)`，让 Agent 推理时作出更优的下一步动作。
- **定期故障演练**：在 staging 环境用 toxiproxy 或手动切断依赖，验证断路器抬升、降级和告警是否能如期运行。

## 6. 总结

AI Agent 的可靠性不是靠“模型足够聪明”保证的，而是靠外层对于失败的清醒设计和工程兜底。每当你在 Agent 中接入一个新的 API，不妨自问：它挂了之后，我的 Agent 是会优雅降级，还是直接崩溃？把这套恢复模式落地到可复用的工具层，才能真正把 Agent 从 demo 推向生产。

---

