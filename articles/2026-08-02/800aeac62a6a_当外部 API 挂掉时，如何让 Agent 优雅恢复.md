---
title: 当外部 API 挂掉时，如何让 Agent 优雅恢复
feedId: 31359
source: 综合讨论
publishedAt: 2026-08-02
---

## 背景：Agent 的“脆弱咽喉”
在 OpenClaw/Agent/MCP 的自动化工作流里，外部 API 调用几乎无处不在——天气查询、翻译接口、对象存储、向量数据库、第三方认证等等。任何一个工具调用都可能因为网络抖动、上游限流、凭证过期、5xx 瞬间失败。如果 Agent 没有针对性的容错设计，轻则单次任务崩掉，重则引发连锁超时，拖垮整个 Pipeline。

更麻烦的是，很多 Agent 插件或 MCP 服务默认只会把 API 报错原封不动抛出，开发者往往在联调阶段才发现：“哦，这里不能直接 fail，得降级给个兜底回答。” 这篇文章就来聊一聊，**在工程上如何为 Agent 构建一套可靠的外部 API 错误恢复机制**，不追求零故障，而是做到“挂了也能优雅收场”。

## 问题拆解：我们需要解决哪几层故障？
常见故障形态可归纳为四类：
1. **瞬时故障**：如网络闪断、临时限流（429）、短暂 503。适合重试。
2. **持续不可用**：上游长时间崩溃或维护，返回 500/502。需要降级。
3. **语义失败**：API 正常返回，但结果不符合预期（例如空数据、解析错误）。需要校验与兜底。
4. **凭证/配置失效**：密钥过期、权限变更。需要尽早发现并告警。

Agent 的错误恢复策略不是“重试三次”就能覆盖的，**必须在工具调用这一层构建一套分层防御**。

## 实践：在 OpenClaw 工具层封装 Resilience
以 OpenClaw 的 MCP 插件开发为例（同样适用于通过 function calling 接入的工具），我们可以在每个外部 API 工具内部统一处理韧性逻辑。下面是一个基于 Python 的实现思路，核心依赖 [`tenacity`](https://github.com/jd/tenacity) 库。

```python
from tenacity import (
    retry, stop_after_attempt, wait_exponential,
    retry_if_exception_type
)
import requests

class ExternalAPITool:
    def __init__(self, max_retries=3, fallback_value=None):
        self.max_retries = max_retries
        self.fallback_value = fallback_value or {"summary": "Service unavailable, using cached hints."}

    @retry(
        stop=stop_after_attempt(3),
        wait=wait_exponential(multiplier=1, min=1, max=10),
        retry=retry_if_exception_type((requests.Timeout, requests.ConnectionError)),
        reraise=True
    )
    def _call_api(self, payload):
        resp = requests.post("https://api.example.com/v1/query", json=payload, timeout=5)
        resp.raise_for_status()
        return resp.json()

    def execute(self, payload):
        try:
            return self._call_api(payload)
        except requests.HTTPError as e:
            if e.response.status_code == 429:
                # 限流场景可加入额外等待或记录告警
                print("Rate limited, falling back.")
            elif e.response.status_code >= 500:
                print("Server error, falling back.")
            return self.fallback_value
        except Exception:
            return self.fallback_value
```

**关键设计点**：
- **重试只用于瞬时错误**：只针对 timeout、connection error，避免对 400 等客户端错误重试。
- **指数退避**：避免重试风暴。
- **兜底返回**：每个工具设计一个 `fallback_value`，比如用缓存的上次结果、静态模板或简化回答。Agent 拿到兜底数据后可以继续向下游提供有限能力的响应。
- **工具契约明确**：在 MCP 工具的 description 里声明可能降级，让 LLM 知道何时该切换策略。

## 踩坑记录
1. **重试无端放大流量**：未区分 4xx 和 5xx，对 400 重试只是浪费上游资源。务必修正 retry 条件。
2. **幂等性缺失**：如果 API 是写操作（如发送消息、创建资源），重试可能导致重复。此时必须要求上游支持幂等 key，或在工具层做去重（如记录请求哈希）。
3. **降级数据过期**：使用缓存的旧数据，可能误导 LLM 生成内容。需要在降级时附加时间戳，并让模型在回答中明确说明数据是基于缓存的。
4. **断路器状态不可见**：复杂场景下直接硬编码重试难以观察。建议引入简单的内存状态，连续失败 N 次后短时间内直接返回兜底，避免无谓尝试。

## 可复用的工程建议
- **在工具注册层统一处理**：不要让每个开发者分别写 try-catch。可以提供一个 `BaseMCPServer` 抽象类，子类只需实现 `_call_api`，超类负责重试、降级和日志。
- **降级策略分级**：划分为“无降级（关键业务）”、“返回空列表”、“返回静态兜底”、“尝试备用 API”四级。配置化驱动。
- **告警不可省**：降级发生时，应通过 webhook/OpenClaw 的监控通道发送事件，避免“静默降级”演变为线上事故。
- **让 LLM 感知降级**：在 MCP 工具返回中增加 `status: degraded` 字段，并在工具描述里告知模型可以根据状态调整措辞，比如“当前服务不稳定，以下为上次可用的数据”。

## 总结
为 Agent 构建错误恢复，本质是**在不可靠的外部依赖之上铺设一层可预期的韧性层**。不是追求永不挂，而是确保挂掉时系统仍能以有限但可控的方式继续运转。OpenClaw/MCP 的插件架构本身就适合做这类横向切片：把重试、降级、告警内聚在每一个工具实现里，Agent 只需面对一个“要么返回有效数据，要么返回降级结果”的一致界面。少一点脆断，多一点工程化的温柔，这样的 Agent 才能在真实环境中跑得久。

---

