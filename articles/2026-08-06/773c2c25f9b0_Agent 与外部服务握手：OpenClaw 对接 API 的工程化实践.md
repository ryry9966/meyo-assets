---
title: Agent 与外部服务握手：OpenClaw 对接 API 的工程化实践
feedId: 31854
source: 综合讨论
publishedAt: 2026-08-06
---

# Agent 与外部服务握手：OpenClaw 对接 API 的工程化实践

## 背景

LLM Agent 最薄弱的环节往往是“不知道现实世界正在发生什么”。训练数据有截止时间，而用户问“今天杭州会不会下雨”或者“帮我查一下订单状态”时，Agent 必须实时访问外部服务。这就需要一次可靠的握手：让 Agent 通过工具（Tool）去调用 API，再把结构化结果喂回推理链路。

OpenClaw 原生支持 MCP（Model Context Protocol）和本地函数注册两种方式将外部服务暴露为 Agent 可用的工具。本文章不讨论协议选型，而是聚焦工程侧的真实问题：对接一个外部 API 时，如何做得**安全、稳定、可观测**，而不是写一堆一次性脚本。

## 问题拆解

假设我们要让 OpenClaw 的 Agent 查询某个城市的实时天气。这个任务看起来简单，但隐藏了以下几个坑：

1. **API 鉴权**：Key 如何存储，如何不泄露到 Prompt 或日志里。
2. **参数校验**：Agent 给出的输入可能带有额外空格、缩写、非标准地名。
3. **响应体积**：第三方 API 可能返回几十 KB 的 JSON，直接把原始响应塞进上下文会快速耗尽 Token 预算。
4. **失败处理**：网络超时、API 限流、返回格式突变，不能让 Agent 直接看到堆栈信息。
5. **可测试性**：总不能每次调试都去真的调用外部 API。

下面我们用 OpenClaw 的本地工具注册方式演示一个工程化对接流程，MCP 工具也可以通过类似的思想做适配层。

## 实践步骤

### 1. 定义工具 Schema
在 OpenClaw 中，工具就是一个带有明确输入 schema 和描述的函数。工具描述（description）会直接进入 Prompt，影响模型调用的准确性。要写得像 API 文档一样精准。

```python
import json
import os
import httpx
from openclaw.tools import tool
from pydantic import BaseModel, Field

class WeatherInput(BaseModel):
    city: str = Field(..., description="城市名称，使用中文全称，例如'杭州'、'北京'")

@tool(
    name="get_weather",
    description="查询指定城市当天实时天气。返回温度、湿度、天气状况摘要。仅支持中国城市。",
    input_schema=WeatherInput
)
async def get_weather(city: str) -> str:
    api_key = os.getenv("WEATHER_API_KEY")
    if not api_key:
        return "错误：天气服务未配置。"
    ...
```

**要点：** 输入 schema 的类型约束和文字描述直接决定了模型会不会乱填。`city` 用 `str` 并给出示例，能显著降低无效调用。

### 2. 封装 API 调用，控制安全边界
把真实 HTTP 请求包在一个 async 函数里，加上超时、异常捕获与响应裁剪。不要用 `requests`，它在异步工具里会阻塞事件循环，用 `httpx` 或 `aiohttp`。

```python
async def _fetch_weather(city: str) -> dict:
    url = "https://api.weather.example.com/v1/now"
    params = {"city": city}
    headers = {"Authorization": f"Bearer {os.getenv('WEATHER_API_KEY')}"}
    
    async with httpx.AsyncClient(timeout=5.0) as client:
        resp = await client.get(url, params=params, headers=headers)
        resp.raise_for_status()
        data = resp.json()
    
    # 裁剪：只保留 Agent 需要的字段
    return {
        "city": data.get("city"),
        "temp_c": data.get("temperature"),
        "humidity": data.get("humidity"),
        "condition": data.get("weather")[0].get("description", "未知")
    }
```

**注意：** API Key 只能从环境变量注入，不能在 schema 里出现，也不能作为 `get_weather` 的参数暴露给模型，否则一旦 Prompt 被日志记录，Key 就泄露了。

### 3. 工具函数内部做“翻译”
API 调用成功后，不要返回 raw JSON 字符串，而是按需转为自然语言摘要或结构清晰的 JSON，方便模型二次加工。

```python
@tool(...)
async def get_weather(city: str) -> str:
    try:
        weather = await _fetch_weather(city)
        return json.dumps(weather, ensure_ascii=False)
    except httpx.TimeoutException:
        return "错误：天气查询超时，请稍后重试。"
    except httpx.HTTPStatusError as e:
        if e.response.status_code == 429:
            return "错误：天气服务繁忙，请等待一分钟再试。"
        return "错误：天气数据暂时不可用。"
    except Exception:
        # 绝不泄漏内部异常
        return "错误：查询失败。"
```

**理由：** 返回给 Agent 的内容会被注入到上下文，如果直接抛出 Python 堆栈，模型可能会“幻觉”出奇怪的解决方案，或者把错误信息编造成事实。

## 踩坑记录

1. **响应尺寸吞噬 Token**  
   典型的第三方 API 会返回大量冗余字段、广告数据。某次接入一个开放地图 API，原始响应 12KB，经过裁剪只剩 200 字节。不做裁剪，三轮对话就能把 GPT-4 的 8K 窗口撑爆。建议强制裁剪，宁可多剪掉一些元数据。

2. **工具调用无限循环**  
   如果 Agent 拿到的错误信息是自然语言“错误：查询失败”，它有时会再次调用同一个工具试图纠正。可以在工具返回的错误信息中加入“请尝试其他方式”或直接告诉 Agent 放弃该工具。也可以在 OpenClaw 侧设置相同工具的最大连续调用次数（通常 2 次就够了）。

3. **网络超时导致 Agent 挂起**  
   OpenClaw 的工具调用默认有一个总超时，但如果外部 API 偶发变慢，会直接影响用户体验。给每个工具单独设超时（如 5 秒），比全局超时更灵活。

4. **真实环境不可测**  
   开发阶段频繁调用外部 API 会被限流，也会产生费用。实现一个简单的本地 mock 工具，通过环境变量 `USE_MOCK_API=true` 返回静态数据，能让调试效率大幅提升。

## 可复用建议

基于多次对接外部服务的经验，可以提炼出一个通用 `APIToolWrapper` 模式：

- **统一请求器**：用 `httpx.AsyncClient` 单例管理连接池、默认超时和重试（用 `tenacity` 库实现退避重试）。
- **响应管道**：定义 `extract_fields(response_json)` 方法，每个 API 只需写裁剪逻辑。
- **错误映射**：维护一个状态码到用户友好错误的字典，禁止直接透传上游错误。
- **日志记录**：记录每次工具调用的参数、耗时、裁剪后大小，但不记录 API Key 和完整响应体，以便后续排查。
- **缓存**：对时效性要求不高的数据（如天气可以缓存 10 分钟），用简单的内存 TTL 缓存减少 API 调用。

把这些模式沉淀成内部库，再对接新的 API 就只需写 Schema 和字段提取逻辑。

## 总结

Agent 对接外部服务，本质上是在大模型的推理循环里插入一个不可靠的 I/O 调用。工程化的重点不是能跑通，而是控制这个 I/O 的**半径**：任何一次外部调用都不该导致 Agent 崩溃、泄露凭证、耗尽 Token 或给出不可信的答案。

在 OpenClaw 生态里，无论是本地工具还是 MCP 插件，这套做法都适用：定义清晰的 Schema，封装安全的 HTTP 层，做激进的响应裁剪，用友好的错误信息引导 Agent 优雅退出。做到这些，你才能说 Agent 真的和外部世界“握手”了，而不是随便拉了根线。

---

