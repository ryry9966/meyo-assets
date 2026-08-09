---
title: AI 助手的 heartbeat 设计：轮询 vs 推送的工程取舍
feedId: 32269
source: 综合讨论
publishedAt: 2026-08-09
---

## 背景

在搭建基于 OpenClaw 的 Agent 或 MCP 工具链时，我们经常需要回答一个看似简单的问题：助手还活着吗？无论是长时间运行的自动化流水线、用户侧插件，还是 MCP 服务器与客户端之间的连接，一旦缺乏有效的心跳检测，就可能出现任务卡死、资源泄漏或状态不一致。心跳（heartbeat）设计正是这类问题的第一道防线。

社区中常见两种实现方向：客户端定时**轮询**（polling）和服务端主动**推送**（push）。这两者没有绝对的优劣，但在不同的 Agent 架构和部署环境下，取舍会显著影响系统的延迟、资源消耗和可维护性。本文不搬运泛热点，只复盘我们在 OpenClaw 实践中的真实考量与踩坑。

## 问题定义

以 MCP 场景为例：一个 OpenClaw Agent 通过 `stdio` 或 HTTP 传输连接到一个远端 MCP 服务器，提供工具调用能力。我们希望 Agent 能实时感知连接是否正常，并在断连后及时重试或告警。类似需求也出现在 Agent 的长时间后台任务中——调度器需要知道 Worker 是否还在工作。

我们需要确定三点：
1. 心跳由哪一端发起？
2. 间隔多大合理？
3. 超时后如何恢复？

## 轮询方案及其代价

最简单的实现是客户端每隔 N 秒向服务端发送一个轻量请求（如 `GET /health` 或专门的 `ping` 工具调用）。在 OpenClaw 插件或 Agent 循环中，这不过是一个定时任务。

```python
import asyncio

async def poll_heartbeat(url, interval=5, timeout=30):
    last_ok = time.time()
    while True:
        try:
            async with session.get(url, timeout=5) as resp:
                if resp.status == 200:
                    last_ok = time.time()
        except Exception:
            if time.time() - last_ok > timeout:
                raise ConnectionError("Heartbeat lost")
        await asyncio.sleep(interval)
```

**优点：** 逻辑直白，无状态，与现有 HTTP 基础设施兼容良好，不需要长连接支持。  
**缺点：**
- 存在检测盲区，最大延时 = 轮询间隔 + 超时阈值。如果 Agent 需要秒级感知，轮询频率必须提高，直接抬高 CPU 和网络开销。
- 服务端无状态，难以对“死”客户端主动释放资源。
- 在分布式或负载均衡后面，多个实例可能同时轮询，产生惊群效应。

在早期原型中，这个方案确实能跑通，但随着 Agent 数量增加和时延要求变严，我们遇到了明显的资源瓶颈。

## 推送方案：SSE 与 WebSocket

更为实时的做法是让服务端主动推送心跳，客户端仅需监听。这通常借助持久连接实现，例如 Server-Sent Events（SSE）或 WebSocket。

以 SSE 为例，在 FastAPI 中构造一个心跳端点：

```python
from fastapi import FastAPI, Request
from sse_starlette.sse import EventSourceResponse
import asyncio

app = FastAPI()

@app.get("/heartbeat-sse")
async def heartbeat_sse(request: Request):
    async def event_generator():
        while True:
            if await request.is_disconnected():
                break
            yield {"event": "heartbeat", "data": "alive"}
            await asyncio.sleep(5)
    return EventSourceResponse(event_generator())
```

OpenClaw Agent 侧可以借助 `httpx-sse` 或浏览器原生 `EventSource` 接收事件，并运行内部定时器：如果在 2 个心跳周期内未收到事件，则判定连接丢失，触发重连。

WebSocket 的思路类似，使用 `ping/pong` 帧更高效，但需要在中间层（如 Nginx、API Gateway）上额外配置支持。

**推送的优势：**
- 实时性高，故障检测延时仅取决于网络与本地超时判断，可做到秒级。
- 服务端可以精确感知每个客户端的活跃状态，方便主动清理半开连接。
- 单条长连接，避免频繁建连/断连的开销。

**代价与复杂度：**
- 长连接给基础设施带来考验：Nginx 的 `proxy_read_timeout`、Kubernetes Ingress 的超时、云 LB 的空闲断开等，都需要仔细调整。
- 后端需要维护连接映射和会话状态，增加内存占用。
- 客户端必须实现健壮的重连逻辑（指数退避、jitter），否则断线后可能长时间无法恢复。

## 踩坑与经验

**坑 1：代理与负载均衡的超时杀连接**  
在线上 Kubernetes + Nginx Ingress 环境中，SSE 连接经常在 60 秒后被无故切断。排查发现是 `proxy_read_timeout` 默认值作祟。将 `location` 块中的超时调大至 300s，并设置 `proxy_buffering off;` 解决问题。如果是云厂商的 LB，还需要确认其空闲超时配置。

**坑 2：客户端“假死”检测**  
单纯依赖 SSE 流中的 `heartbeat` 事件，如果服务端协程阻塞（比如数据库查询卡死），事件也不会发出。因此我们在 Agent 端增加了**兜底轮询**：如果 3 个推送周期内未收到任何事件，会回退到一个低频率的 `/health` 轮询作为最终确认。这种混合模式避免了误报。

**坑 3：心跳与业务消息的干扰**  
在 MCP JSON-RPC 场景中，如果使用 WebSocket 同时传输心跳和工具调用，客户端需要用不同的超时逻辑处理。我们将心跳 `ping` 和业务请求分成不同层：心跳只用于连接存活性，业务超时由上层重试机制负责，避免相互污染。

**坑 4：间隔设置的艺术**  
太频繁会对服务端形成高频压力，太疏则失去实时性。一个可用的经验值是：心跳间隔 = 预期恢复时间 / 3。例如允许 30 秒内发现断开，则间隔设为 10 秒，超时判定为 20 秒。务必加入随机抖动，防止“雷鸣群效应”。

## 可复用建议

根据我们的实践，可以将 heartbeat 模块抽象为一个可配置的组件，供 OpenClaw 插件或 MCP 连接器直接使用：

1. **传输抽象**：定义 `HeartbeatTransport` 接口，分别实现 PollingTransport、SSETransport、WebSocketTransport。
2. **熔断与重连**：内置指数退避和状态机（alive、suspect、dead），并暴露回调事件，便于上层执行资源清理或告警。
3. **混合探测**：默认使用推送方式，当推送连续失败 N 次后自动降级为轮询确认，提升抗干扰能力。
4. **基础设施适配文档**：为常见部署环境（Nginx、Kong、AWS ALB）提供推荐配置片段，避免使用者反复踩坑。

在 OpenClaw 社区中，我们已将这一模式沉淀为 `openclaw-heartbeat` 小工具，支持通过 `plugin.json` 中声明 `heartbeat` 字段启用，内部灵活切换传输方式，极大降低了重复开发成本。

## 总结

AI 助手的心跳设计没有银弹。**轮询**是实现成本最低、适用性最广的初版方案，适合低并发、宽松时效的场景。**推送**则是面向实时性要求较高、需要精细管理连接生命周期的进阶选择。实际工程中，我们更推荐采用“推送为主、轮询兜底”的混合策略，并做好超时配置、重连机制和监控告警。一个小小的心跳，往往能暴露出一整条链路的健壮性短板——这也正是工程化的乐趣所在。

---

