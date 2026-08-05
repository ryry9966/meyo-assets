---
title: 让 Agent 能调用真实 API：OpenClaw 外部服务对接与 MCP 工具化实践
feedId: 31817
source: 综合讨论
publishedAt: 2026-08-06
---

## 背景：Agent 为什么要“握手”外部 API

如果 Agent 只靠预训练数据和内部知识库回答问题，它很快会遇到两个天花板：**实时性缺失**和**动作能力缺失**。  
用户问“今天北京会不会下雨”，Agent 需要查询实时天气；用户说“帮我把这条告警静默 30 分钟”，Agent 需要调监控平台的 API。这些场景下，Agent 不再是一个对话模型，而是一个编排器——它必须能准确、稳定地调用外部服务。

在 OpenClaw 生态里，Agent 与外部 API 的“握手”通常通过**工具调用（Tool / Function calling）**实现，而 MCP（Model Context Protocol）则是目前最通用的工具接入协议。这篇文章就从工程落地的角度，拆解如何用 MCP 把任意 REST API 变成 Agent 可用的工具，并给出真实踩坑记录和可复用的构建建议。

## 问题：对接外部 API 不止是写个 HTTP 请求

表面上看，对接就是写一个 Python 函数，里面发个 requests.get，然后让 Agent 调用。但在实际运行中，你会碰到一连串工程问题：

- **认证信息怎么安全注入？** API Key、OAuth Token 不能写死在代码或工具描述里。
- **工具描述写多细，Agent 才会准确调用？** 太简略，Agent 不知道该传什么参数；太啰嗦，可能反而误导。
- **API 超时、限流、返回非 200 时，Agent 怎么处理？** 直接把 500 堆栈丢给用户是灾难。
- **工具返回结果太大或非结构化，如何对齐 Agent 的上下文窗口和推理逻辑？**
- **如何在不修改 Agent 核心逻辑的前提下，把一个外部服务快速接入并实现可插拔？**

下面我们用 OpenClaw + 自定义 MCP 服务器，跑通一套“从 API 到 Tool”的标准化路径。

## 做法：用 MCP 把 REST API 封装为可调用工具

以下步骤以 Python 和 `fastmcp` 库为例，将 OpenWeatherMap 的天气 API 封装成一个 `get_weather` 工具。

### 1. 编写 MCP 服务器

新建 `weather_server.py`：

```python
import os
import httpx
from fastmcp import FastMCP

mcp = FastMCP("Weather")

API_KEY = os.getenv("OPENWEATHER_API_KEY")
BASE_URL = "https://api.openweathermap.org/data/2.5/weather"

@mcp.tool()
async def get_weather(city: str, country: str = "") -> str:
    """
    查询指定城市的实时天气，返回温度、湿度、天气描述。
    city: 城市英文名 (如 Beijing)
    country: ISO 3166 两位国家代码 (如 CN)，可选
    """
    if not API_KEY:
        return "错误：未配置天气 API 密钥。"

    params = {"q": f"{city},{country}" if country else city,
              "appid": API_KEY, "units": "metric", "lang": "zh_cn"}
    async with httpx.AsyncClient(timeout=10.0) as client:
        try:
            resp = await client.get(BASE_URL, params=params)
            resp.raise_for_status()
            data = resp.json()
            weather = data["weather"][0]["description"]
            temp = data["main"]["temp"]
            humidity = data["main"]["humidity"]
            return f"{city}天气：{weather}，温度 {temp}°C，湿度 {humidity}%"
        except httpx.HTTPStatusError as e:
            if e.response.status_code == 404:
                return f"未找到城市 '{city}' 的天气信息，请检查城市名。"
            return f"天气服务暂时不可用 (HTTP {e.response.status_code})。"
        except httpx.TimeoutException:
            return "天气服务请求超时，请稍后重试。"
```

要点：
- 用环境变量注入 `API_KEY`，永远不写死在代码里。
- 函数 docstring 就是给 Agent 看的工具描述，必须明确**功能、参数含义和类型**，不用技术黑话。
- 全部异常被捕获并翻译成**自然语言结果**，Agent 会直接把这些文字回复给用户。

### 2. 在 OpenClaw 中接入 MCP 服务器

OpenClaw 支持通过配置文件加载 MCP 工具服务器。在 `openclaw.yaml`（或你项目的 Agent 配置）中添加：

```yaml
mcp_servers:
  weather:
    command: python
    args: ["weather_server.py"]
    env:
      OPENWEATHER_API_KEY: ${OPENWEATHER_API_KEY}
```

重启 OpenClaw 后，`get_weather` 就会出现在当前 Agent 的工具列表中。在系统 prompt 里可以加上一行引导：“当用户询问天气时，使用 get_weather 工具获取信息。”

### 3. Agent 调度与工具返回

当用户问“今天北京天气如何”，OpenClaw 会把这句话和工具定义一起发给 LLM，LLM 规划出工具调用 `get_weather(city="Beijing")`，OpenClaw 通过 MCP 执行并拿到结果字符串，再继续生成最终回答。

## 踩坑与排障

1. **工具描述不精确导致调用失败**  
   如果 `city` 参数描述里只写“城市名”，Agent 可能直接传“北京”。OpenWeatherMap 需要英文名，我们可以在描述里明确“城市英文名”，或再封装一层中文到英文的映射。也可以在工具内部做一层模糊翻译，但会增加复杂度。最佳实践是工具描述与 API 所需格式保持一致。

2. **环境变量未正确注入**  
   MCP 服务器启动时读不到 `API_KEY`，工具调用会直接返回“未配置密钥”。检查 OpenClaw 是否把 `env` 里的变量正确传递到子进程，必要时先用 `print(os.environ.get(...))` 调试。

3. **工具超时和重试**  
   HTTP 客户端设置的 10 秒超时对于某些慢 API 可能太紧。可以引入指数退避重试，但注意：Agent 调用链的总超时通常由上游控制，重试过多会导致 Agent 响应延迟。建议在工具层最多重试一次（例如用 `tenacity` 库），并将最终超时异常转化为友好提示。

4. **工具返回的数据量**  
   一般 API 可能会返回完整 JSON 几十 KB，直接让 LLM 读容易超过上下文或推理偏差。务必在工具里**裁剪和归整**，只抽取回答用户问题所必需的关键字段，并形成简短的自然语言摘要。

5. **流式调用下的时序问题**  
   如果 OpenClaw 在流式生成过程中触发工具调用，要确保工具返回的完整内容准确拼接到下一轮推理上下文。一些早期版本可能存在截断，升级到最新 OpenClaw 一般可解。

## 可复用的工程化建议

- **将 API 接入抽象为“适配器”模式**  
  一个 MCP 服务器可以暴露多个工具，每个工具对应一类 API 操作。别把一堆无关接口塞在同一函数里。比如监控系统可以拆成 `silence_alert`、`list_alerts` 两个工具，描述清晰。
- **返回格式统一为 JSON 结构化的自然语言**  
  不是让工具直接返回 JSON 字符串，而是用 JSON 组织关键信息后再转成一段文本，方便 LLM 二次总结，也方便日后在 MCP 层面做结构化监控。
- **加入调用日志和审计**  
  在工具函数中打印（或写日志）每次调用的参数、返回摘要和耗时。出现问题时，你可以立刻判断是 API 挂了还是 Agent 传错了参数。
- **API 凭证统一用配置管理**  
  通过环境变量、Secrets 文件或 OpenClaw 的 `env` 注入，切忌写入工具描述或源码。如果你需要对接多个环境（开发/生产），用不同 MCP 配置分别加载。
- **工具测试与模拟**  
  不要每次都在 Agent 对话里调试工具。用 MCP Inspector 或简单的 Python 脚本直调工具函数，确保 API 通路正确后，再交给 Agent 调度。

## 总结

Agent 对接外部服务，本质上就是把“不确定的自然语言”到“确定的接口调用”这条链路做健壮。  
OpenClaw + MCP 给出的方案很清晰：**把每个外部 API 封装成可描述的、独立测试的工具，并用统一的协议接入 Agent。** 关键是做好异常保护、返回裁剪和凭证管理，剩下的就交给 Agent 去编排。

当我们把更多真实 API 接入 Agent 的工作空间后，Agent 就不再是聊天的玩具，而是一个能读、能写、能操作外部系统的自动化节点。这，才是 Agent 生产力的真正开始。

---

