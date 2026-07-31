---
title: AI Agent 错误恢复实战：当外部 API 挂了怎么做降级与重试
feedId: 31086
source: 综合讨论
publishedAt: 2026-07-31
---

## 背景：没有永远可用的 API

在 OpenClaw 的 Agent 实践中，几乎所有有意义的自动化都依赖外部 API：搜索、网页抓取、数据库查询、天气预报、翻译服务…… 这些依赖一旦不可用，Agent 的整个任务链就可能断裂。更糟的是，Agent 通常运行在一个无人工干预的循环里，如果对失败没有设计，轻则死循环，重则静默失败，用户只收到一个空荡荡的回复。

MCP（Model Context Protocol）和各类插件让工具接入变得更简单，但**只管“接入”不管“失败”，是 90% 的生产事故根因**。本文从工程化角度梳理一套轻量错误恢复方案，不引入额外服务，只在工具层和 Agent 指令层做文章。

## 问题：Agent 面对 API 异常时的三种典型坏行为

1. **快速失败，吐一堆堆栈给用户**  
   Agent 调用的工具直接抛出未捕获异常，LLM 要么原样输出错误信息，要么不知所措，把晦涩的 traceback 塞进回复。

2. **无脑重试，耗尽资源**  
   工具内部或 Agent 的思考循环里写了“再试一次”，但没有退避、没有上限。一个服务挂了 10 分钟，Agent 可能已经重试了上千次，Token 和请求配额瞬间打光。

3. **静默吞错，产生幻觉**  
   更隐蔽：工具把错误藏掉，返回空结果或上一次的缓存。Agent 以为拿到了真实数据，继续推理，最终输出一个逻辑自洽但完全脱离现实的答案。

这三种情况的共同后果是：用户信任崩塌，生产环境根本不敢长挂 Agent。

## 做法：三层恢复体系

我们采用“工具层降级 + 指令层兜底 + 运行时熔断”的组合，不侵入业务代码太深，适合 MCP 工具开发者和 Agent 配置者直接使用。

### 第一层：工具端 - 重试与降级一体

每一个工具函数都应该有一个包装层，处理三类异常：瞬时错误（网络抖动）、服务错误（503）、参数错误（4xx）。以 Python 的 MCP server 工具为例：

```python
import asyncio
import random
from typing import Any, Callable
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("my_tools")

def with_fallback(fallback_result: Any = None):
    def decorator(func: Callable):
        async def wrapper(*args, **kwargs):
            max_retries = 3
            base_delay = 1.0
            last_exc = None
            
            for attempt in range(max_retries):
                try:
                    return await func(*args, **kwargs)
                except TemporaryError as e:
                    last_exc = e
                    delay = base_delay * (2 ** attempt) + random.uniform(0, 0.5)
                    await asyncio.sleep(delay)
                except FatalError:
                    raise  # 参数错误等不应重试
            
            # 所有重试用尽，执行降级
            if fallback_result is not None:
                return fallback_result
            raise last_exc or RuntimeError("Unknown error after retries")
        return wrapper
    return decorator
```

这里刻意区分了 `TemporaryError` 和 `FatalError`。搜索 API 的超时（网络瞬断）重试有意义，而 API Key 错误或 403 则应立刻抛出让 Agent 知道。

**降级策略按工具的性质选择：**
- **只读查询类**（天气、汇率）：返回一个带 `status: "degraded"` 的结构，并附上最近一次的有效缓存或友好提示。  
- **动作类**（发送消息、创建工单）：绝不虚构成功，应明确返回错误信息，由 Agent 转告用户，或放入延迟队列。
- **核心路径**：如果工具是整个流程的必须环节，降级结果要带上 `retry_after` 建议，让上层决策。

### 第二层：Agent 指令 - 教会模型理解服务降级

在 Agent 的 system prompt 或 MCP 工具描述中加入错误协议说明，例如每个工具在失败时都会返回一个字段 `error` 和 `severity`（`temporary`/`permanent`）。在 OpenClaw 的场景里，可以直接在工具函数描述中写明：

> 如果返回的 JSON 中包含 `"error": true`，请根据 `suggestion` 字段给用户解释当前发生了什么，并告知可尝试的操作，不要自行解读原始报错。

同时，可以在全局指令中加一条：“当连续两个工具调用返回临时性错误时，暂停当前任务，向用户报告状况，并询问是否继续等待或切换备选方案。”

这样做的好处是，模型不会因为得不到数据就开始编造，而是懂得“承认不知道”也是一种有价值的输出。

### 第三层：运行时熔断与监控（可选但推荐）

如果 Agent 运行在长时间无人盯守的环境下，可以在工具调用入口做一个简单的滑动窗口熔断器：同一个工具 5 分钟内失败超过 10 次，直接对该工具短路 3 分钟，返回降级响应。这可以防止突发的 API 故障把整个 Token 预算烧穿。

在 OpenClaw/Node.js 环境下，可以用一个内存 `Map` 记录，实现不超 30 行的熔断逻辑。

## 踩坑点

1. **把“重试”写在 LLM 的思考循环里**  
   看到太多人让 Agent 自己决定“再试一次”，结果就是工具明明返回了错误，Agent 却在说“也许我拼错参数了，让我再试一次”，进入无意义的自我纠正循环。**重试必须在工具层完成，不要让 LLM 决策重试。**

2. **降级返回了空列表，却没说这是降级**  
   比如搜索工具不可用，直接返回 `[]`。Agent 会认为“没有相关信息”并告诉用户不存在，而实际上可能搜一下就有。所以降级结果必须携带明确的可区分标记，如 `"fallback": true`。

3. **超时不设限**  
   一个 API 调用卡住 30 秒，没有设定 timeout，Agent 就被挂起。所有外部 API 调用必须设定严格的 connect timeout 和 read timeout（如 5s/15s）。超时异常应归类为 TemporaryError 并进入重试逻辑。

4. **错误日志只输出到控制台**  
   出问题之后靠 grep 日志找原因，效率极低。建议为每个工具调用生成一个 `trace_id`，并在工具返回里附上，这样用户报告问题时可以直接定位。

## 可复用建议

- **为 MCP 工具制定统一的错误返回 schema**，团队内所有工具都遵守。至少包含 `error: bool`、`message: str`、`retryable: bool`、`trace_id: str`。
- **写工具时先用装饰器/中间件处理异常**，不要在每个函数里写 try/catch。装饰器方式让正常逻辑保持干净。
- **在 Agent 配置中增加一个 `tools/error_handling` 文档**，说明每个工具可能的错误模式及应对示例，作为 prompt 的一部分。
- **主动测试故障场景**：定期在非生产环境中用 chaos 方法关掉某个 API，观察 Agent 的行为，持续优化降级提示。

## 总结

AI Agent 的错误恢复不是锦上添花，而是能否从“玩具”走向“工具”的分水岭。核心思路就三句话：**工具层扛住瞬时失败并自主降级，指令层教会模型读懂服务状态，运行时用轻量熔断保护资源**。这三层组合下来，不依赖任何外部组件，就能让 Agent 在 API 颠簸中保持体面，至少不制造幻觉、不烧预算、不丢失用户信任。

在 OpenClaw 和 MCP 生态里，工具和 Agent 指令都是我们直接可控的，花一个下午把这套体系落地，未来每一个自动化流程都会更稳。

---

