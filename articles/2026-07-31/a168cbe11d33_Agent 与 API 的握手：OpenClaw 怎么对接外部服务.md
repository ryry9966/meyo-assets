---
title: Agent 与 API 的握手：OpenClaw 怎么对接外部服务
feedId: 31101
source: 综合讨论
publishedAt: 2026-07-31
---

# Agent 与 API 的握手：OpenClaw 怎么对接外部服务

## 背景
很多同学在用 OpenClaw 构建 Agent 时，都会遇到同一个问题：Agent 只能基于训练数据和对话上下文推理，没办法直接查库存、发邮件、调 Jenkins。想让 Agent 真正干活的，就得给它装上“手”——通过工具（tool）去调用外部 API。

和其他框架不同，OpenClaw 不强依赖 MCP 协议，它原生支持将 Python 函数直接暴露为工具，并通过模型自身的 function calling 能力完成调用。这让我们可以把任意 REST/gRPC 服务封装成工具，再交给 Agent 调度，实现“Agent 与 API 握手”。

这里记录一次将企业内部“工单查询”API 接入 OpenClaw 的完整过程，包含方案设计、编码步骤、踩坑和可复用模板。

## 问题
目标：让 OpenClaw Agent 能回答“我最近三个工单的状态是什么？”。  
约束：
- 工单系统提供 REST API（`GET /api/v1/tickets?owner={user}`，Bearer Token 鉴权）
- 调用需带上当前用户的身份标识（从会话上下文获取）
- 不允许将 API Token 明文写在 prompt 或函数签名里
- 需要处理超时、重试和友好错误提示

## 做法 / 步骤
### 1. 设计工具函数
在 OpenClaw 里，工具就是一个普通的 async 函数，配合类型注解和 docstring 即可。我们需要一个 `get_my_tickets` 工具，签名如下：

```python
async def get_my_tickets(limit: int = 5) -> list[dict]:
    """获取当前用户的工单列表。limit: 返回条数，默认5"""
```

但这里有个关键问题：用户身份从哪里来？OpenClaw 的 runtime context (`openclaw.context`) 可以传递会话信息。我们可以把 user_id 和 auth token 放进 context。

### 2. 注册工具并注入上下文
```python
from openclaw import tool, Context

@tool
async def get_my_tickets(limit: int = 5, ctx: Context = None) -> list[dict]:
    user_id = ctx.session.get("user_id")
    token = ctx.secrets.get("ticket_api_token")
    if not user_id or not token:
        return [{"error": "缺少用户身份或凭证"}]

    url = f"https://ticket.internal.example.com/api/v1/tickets"
    headers = {"Authorization": f"Bearer {token}"}
    params = {"owner": user_id, "limit": limit}

    async with httpx.AsyncClient(timeout=10) as client:
        resp = await client.get(url, headers=headers, params=params)
        resp.raise_for_status()
        return resp.json()["items"]
```

在创建 Agent 时，将工具传入，并在启动时把 `user_id` 和从安全存储读取的 token 塞入会话 context。

### 3. 对接 OpenClaw Agent
```python
agent = OpenClawAgent(
    tools=[get_my_tickets],
    llm="gpt-4o",
    system_prompt="你是一个帮助查询内部工单的助手。"
)

# 每次用户请求前填充上下文
session_context = {
    "user_id": "zhangsan",
    "secrets": {"ticket_api_token": os.getenv("TICKET_API_TOKEN")}
}
response = await agent.run("我最近的工单有哪些？", session_context=session_context)
```

Agent 会自动判断需要调用 `get_my_tickets`，并传入合适的参数（这里 limit 可以让模型自己决定或用户指定）。

## 踩坑点
1. **超时与重试**
   Agent 调用工具时，默认的超时可能太短（尤其内网服务偶尔抖动）。我在 `httpx.AsyncClient` 设置了 10s 超时，并在捕获 `httpx.TimeoutException` 后返回友好 JSON，让模型知道“查询超时，请稍后重试”。不要直接抛出异常导致对话中断。

2. **返回数据过大**
   工单列表可能包含大量字段（描述、评论等），一次返回全部会撑爆上下文窗口，token 消耗剧增。需要在函数里做字段裁剪，只返回 id、标题、状态、创建时间等关键信息，或者增加 `fields` 参数让 Agent 按需选择。

3. **序列化陷阱**
   工具返回值必须是可 JSON 序列化的 dict/list。类似 `datetime` 对象要转成字符串，否则 function calling 回传时会报 `type_error`。

4. **凭证安全**
   绝对不能把 API Token 放在工具的默认参数或环境变量直接暴露给 Agent。OpenClaw 的 `Context.secrets` 是个较好的隔离方案，也可以在外部通过 vault 读取后注入，只给工具函数使用。

5. **函数命名与描述模糊**
   工具的描述（docstring）直接影响模型调用准确率。描述里要明确说明参数含义、返回结构，不然 Agent 可能乱填 limit=100 或者不传 owner。

## 可复用建议
- 将外部服务封装为“领域工具包”（例如 `ticket_tools.py`），统一管理 base URL、认证和错误处理逻辑。
- 使用 Pydantic 模型定义返回结构，既保证序列化，又能自动生成 schema 帮助模型理解（OpenClaw 支持 TypedDict/Pydantic 作为工具返回类型）。
- 为工具添加轻量缓存（例如 `@lru_cache` 或 Redis），当 Agent 短时间内重复调用相同参数时可快速返回，减轻后端压力。
- 记录每次工具调用的输入、输出和耗时，方便调试和审计。OpenClaw 的上下文里可以挂载 logger。
- 提供一个“健康检查”工具（如 `health_check`）让 Agent 在前置步骤确认服务可达，避免连续调用失败浪费 tokens。

## 总结
Agent 与外部服务的对接没有捷径：把 API 封装成安全、可观察、容错的工具，再交给 Agent 调度。OpenClaw 的 tool 定义方式和上下文机制让这件事变得轻量且可控，相比强制走 MCP 代理的方案更适合内部异构系统的快速接入。实际落地时，把 80% 的精力花在工具的设计与防御性编程上，剩下的 20% 才是 Agent 的编排逻辑。最终得到的是一个既懂自然语言、又能直连生产数据的助手，而不是一个只会聊天的聊天机器人。

---

