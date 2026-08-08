---
title: Agent 与 API 的握手：在 OpenClaw 中构建安全的外部服务对接通道
feedId: 32161
source: 综合讨论
publishedAt: 2026-08-08
---

# Agent 与 API 的握手：在 OpenClaw 中构建安全的外部服务对接通道

## 背景

Agent 的推理能力再强，一旦离开实时数据或业务系统，能做的事就十分有限。查询天气、调用内部订单接口、发送通知，这些能力都不是模型本身具备的，必须通过 API 接入外部服务。在 Agent 框架里，“调用工具”是标准解法，但自建工具如果处理不当，反而会成为整个链路的脆弱点。

OpenClaw 作为一个以 Tool/Plugin 为核心的 Agent 框架，允许开发者把任意外部服务封装成可被 LLM 决策调用的模块。这篇文章不教如何写 Prompt，而是聚焦工程落地：如何稳定、安全地把 API 暴露给 Agent。

## 问题

初看很简单：让 Agent 发一个 HTTP 请求就行了。实际上这条路会踩到一连串坑：

- **鉴权泄露风险**：API Key 通过 Prompt 传递容易被日志、缓存无意泄露。
- **异常吞没**：API 超时或返回错误时，如果直接抛异常，Agent 的推理循环可能直接崩溃，而不是收到友好的错误提示。
- **LLM 对原始 JSON 脆弱**：很多 API 返回大量元数据，直接喂给模型容易造成幻觉或误判。
- **事件循环冲突**：Agent 内部几乎都是异步运行，同步 HTTP 调用会阻塞整个事件循环。

所以我们要做的不是“帮 Agent 请求一次”，而是**建立一个隔离层**，把不可控的外部世界变成可控的结构化工具。

## 做法：封装一个外部 API Tool

以下以封装一个 IP 归属地查询服务（使用 ipinfo.io）为例，展示在 OpenClaw 中的标准流程。OpenClaw 的工具定义约定与常见框架（如 LangChain）类似，但更轻量，核心就是注册一个异步函数。

### 1. 目录与依赖
在 OpenClaw 项目的 `tools/` 目录下新建 `ip_lookup.py`。依赖只有 `aiohttp`。

### 2. 定义工具函数
```python
import os
import aiohttp
from openclaw.tools import tool

IPINFO_TOKEN = os.getenv("IPINFO_TOKEN")

@tool(
    name="ip_lookup",
    description="查询 IP 地址的地理位置和运营商信息。参数 ip 为要查询的 IP 地址。"
)
async def ip_lookup(ip: str) -> str:
    if not ip:
        return "Error: ip 参数不能为空"

    url = f"https://ipinfo.io/{ip}/json"
    headers = {"Authorization": f"Bearer {IPINFO_TOKEN}"} if IPINFO_TOKEN else {}

    timeout = aiohttp.ClientTimeout(total=5)
    try:
        async with aiohttp.ClientSession(timeout=timeout) as session:
            async with session.get(url, headers=headers) as resp:
                if resp.status == 200:
                    data = await resp.json()
                else:
                    error_text = await resp.text()
                    return f"API 返回错误 {resp.status}: {error_text[:200]}"
    except aiohttp.ClientError as e:
        return f"请求失败: {str(e)[:200]}"
    except Exception as e:
        return f"未知错误: {str(e)[:200]}"

    # 只提取对 Agent 有用的字段，避免噪声
    city = data.get("city", "未知")
    region = data.get("region", "未知")
    country = data.get("country", "未知")
    org = data.get("org", "未知")
    return f"{ip} 位于 {city}, {region}, {country}, 运营商: {org}"
```

### 3. 注册工具
在 Agent 初始化时导入这个模块，OpenClaw 会自动发现被 `@tool` 装饰的函数并注入给 Agent。

## 踩坑与经验

### 必须保持完全异步
Tool 函数必须是 `async def`，并且内部使用 `aiohttp` 而非 `requests`。一次同步阻塞就可能导致整个 Agent 的响应延迟甚至超时。同样，不要在里面写入任何 `time.sleep`。

### 错误必须翻译成自然语言
Agent 调用工具后会把返回值（字符串）直接作为上下文。如果这里抛出未捕获异常，OpenClaw 会收到一个工具调用失败信号，并可能中断当前任务。因此**所有异常都要在工具内部被截获**，并返回对模型友好的错误描述，如 `Error: API 请求超时，请稍后重试`。不要原样返回堆栈信息。

### 清洗返回值，别当 JSON 搬运工
原始 API 返回可能包含几十个字段，Agent 不需要全部信息。只提取与任务相关的核心字段，组织成一句或一段清晰的自然语言描述。这能明显降低后续推理的幻觉率。

### 鉴权信息与工具分离
密钥一律通过环境变量传入，绝对不要写在工具的描述或默认参数里。工具的描述会暴露给 LLM，写死 token 就意味着泄露。

### 超时与重试
每个对外请求都要设置合理的超时（通常 5–10 秒）。重试逻辑可以统一在基类里实现，而不是在每个工具里裸写。

## 可复用建议

当工具量增加到 5 个以上时，建议做一次抽象：

```python
class BaseAPITool:
    async def _get(self, url, headers=None, timeout=8):
        # 统一实现重试、日志、错误收敛
        ...
    async def _post(self, url, payload, headers=None, timeout=8):
        ...
```

所有具体工具继承该类，只需关心业务逻辑。这套模式能让团队在 10 分钟内接入一个新的外部 API，且行为一致。

如果对接的服务本身已经提供了 MCP 服务器（比如数据库、文件系统），更推荐直接通过 OpenClaw 的 MCP consumer 接入，而不是重复造轮子。但大量业务系统只有传统 REST API，手写 Tool 封装依然是现阶段最务实的做法。

## 总结

Agent 与外部服务的对接，本质上是在不确定的 LLM 推理和确定性的 API 契约之间加一层适配器。这一层的核心任务只有三个：**屏蔽异常、转译结果、隔离风险**。只要这三点做到了，Agent 就能安全地伸长触角，真正进入生产环境。

不要幻想 LLM 能优雅处理一切 API 的意外情况——把世界整理成工具能理解的格式，才是工程上可控的方式。

---

