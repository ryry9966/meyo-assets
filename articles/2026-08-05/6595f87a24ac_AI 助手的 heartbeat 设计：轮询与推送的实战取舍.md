---
title: AI 助手的 heartbeat 设计：轮询与推送的实战取舍
feedId: 31770
source: 综合讨论
publishedAt: 2026-08-05
---

在 OpenClaw/Agent 自动化场景中，让一个长时间运行的 AI 助手保持“可观测”往往是从 Demo 迈向生产的第一步。heartbeat（心跳）是确认 Agent 存活的最基础信号，但在实现方式上，很多团队会在 **轮询（polling）** 和 **推送（push）** 之间反复纠结。本文从工程实践角度，梳理两种方案的适用边界、具体实现、踩坑记录和可复用建议。

## 1. 背景：为什么需要心跳

无论是基于 OpenClaw 框架的 Agent，还是通过 MCP 挂载了多个工具的自动化实例，一旦进入无人值守的后台任务模式，就需要一个外部监控系统知道它是否“活着”。心跳的基本目的包括：

- 检测进程是否崩溃、死锁或陷入无限循环；
- 触发故障切换或自动重启；
- 为负载均衡、任务调度提供活性依据。

但在实际的网络拓扑下，监控端和 Agent 往往分属不同安全域，甚至 Agent 跑在 NAT 之后。这直接引出了轮询和推送两种通信模式的选择。

## 2. 方案对比与实现步骤

### 2.1 轮询模式：监控端主动查询

**适用场景**：Agent 可以暴露网络端口，监控端能直接访问；对实时性要求不高（秒级延迟可接受）。

**实现步骤**：

1. 在 Agent 内部启动一个轻量 HTTP 服务（如 Python `aiohttp`），暴露 `/health` 端点：
```python
import time
from aiohttp import web

async def health_handler(request):
    return web.json_response({
        "status": "ok",
        "timestamp": time.time(),
        "agent_id": "agent-01"
    })
```
2. 监控端使用定时任务（如 `asyncio.create_task` 或 `APScheduler`）每隔 N 秒请求该端点：
```python
import aiohttp

async def check_health():
    async with aiohttp.ClientSession() as session:
        async with session.get("http://agent:8080/health", timeout=5) as resp:
            data = await resp.json()
            # 记录状态，超时或异常则触发告警
```
3. 根据连续失败次数（如 3 次）判定 Agent 异常。

### 2.2 推送模式：Agent 主动上报

**适用场景**：Agent 位于防火墙后或没有固定 IP，但能主动外连监控端；需要近实时感知 Agent 离线。

**实现步骤**：

1. 监控端启动 WebSocket 服务，维护连接池：
```python
import asyncio
import websockets

connected_agents = {}

async def monitor_handler(websocket, path):
    agent_id = await websocket.recv()  # 首条消息注册 ID
    connected_agents[agent_id] = websocket
    try:
        async for msg in websocket:
            # 处理心跳消息，更新最后活跃时间
            pass
    finally:
        connected_agents.pop(agent_id, None)
```
2. Agent 作为 WebSocket 客户端连接并定期发送心跳帧（或直接利用 WebSocket 的 Ping/Pong）：
```python
async def heartbeat_loop(agent_id, uri="ws://monitor:8765"):
    async with websockets.connect(uri) as ws:
        await ws.send(agent_id)
        while True:
            await ws.ping()
            await asyncio.sleep(10)
```
3. 监控端通过超时检测：若 `time.now() - last_ping` 超过阈值，判定 Agent 失联。

## 3. 踩坑记录

- **轮询频率选择**：过于激进的轮询（如 1 秒）会在 Agent 数量多时放大连接风暴，曾经在实际部署中打满过监控端的文件描述符。建议根据容错时间窗口设定，通常 10~30 秒足够。
- **推送方案的连接恢复**：Agent 重启后若监控端未及时清理旧连接，会出现“僵尸连接”。可以在监控端对每个连接设 TTL，Agent 侧增加指数退避重连。
- **防火墙与 NAT**：推送模式要求 Agent 能出网但监控端需有公网可达端口。如果监控端也内网，可考虑引入 MQTT 等消息中间件作为心跳代理，让双方都做客户端。
- **误判处理**：无论是轮询超时还是推送不达，都需要二次确认机制（如连续失败 N 次），避免因网络瞬时抖动导致误切。
- **Agent 内耗**：若 Agent 主线程卡死但 health 线程仍能响应轮询，会形成“伪存活”。推送方案可以将心跳发放在主事件循环中，卡住即停发，检测更真实。

## 4. 可复用的设计建议

1. **混合架构兜底**：当推送连接中断时，监控端可降级为轮询 Agent 暴露的 `/health` 端点作为二次确认，避免单点误判。
2. **心跳信息扩展**：在心跳包中附加当前任务队列长度、最后一条消息时间等业务指标，让心跳同时承担“轻量健康报表”的角色。
3. **MCP 场景中的心跳**：如果采用 MCP 协议的 Agent，可以复用协议内置的 `ping` 能力，在 `mcp.run()` 循环中定时发送，保持长连接的同时实现心跳，减少额外端口占用。
4. **监控端设计**：建议采用无状态监控 + 外部存储记录活性，方便扩展。例如 Redis 记录 Agent 最后心跳时间，由独立告警 Worker 扫描。
5. **尽量异步非阻塞**：所有心跳 IO 操作必须异步，防止阻塞 Agent 主任务。

## 5. 总结

轮询和推送并非对立，而是适用于不同网络约束和实时性要求的互补方案。**能主动出网就用推送，能直连且对秒级延迟不敏感就用轮询，生产环境尽量让两者共存**。真正决定稳定性的，不是某一种通信模式，而是超时重试策略、连接生命周期管理和误判保护机制。在设计 AI 助手的心跳时，先从 Agent 的网络位置、监控端部署形态出发，画出流量路径图，再选择最省心的组合方式，远比追逐某个“先进”的单项方案更有价值。

---

