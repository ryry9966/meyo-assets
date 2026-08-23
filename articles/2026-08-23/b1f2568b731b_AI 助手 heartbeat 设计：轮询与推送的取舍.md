---
title: AI 助手 heartbeat 设计：轮询与推送的取舍
feedId: 34381
source: 综合讨论
publishedAt: 2026-08-23
---

在给 OpenClaw 写插件或 Agent 时，经常会遇到一个“等待”问题：调用一个异步任务后，助手需要知道任务什么时候完成。比如 MCP 工具触发代码分析、流水线构建、或者等待用户在某个外部系统里完成审批。这个“等待”机制，本质上就是 heartbeat——让助手持续感知外部状态变化。

轮询（polling）和推送（push）是两种基本方案。轮询是助手每隔 N 秒去查一次状态接口；推送是外部系统通过 WebSocket、SSE 或 webhook 主动通知。网上很多教程会直接说“用 WebSocket，实时性好”，但在工程实践里，轮询往往是更稳妥的默认选项，推送则是一笔需要算清楚的账。

## 背景：为什么 heartbeat 在 Agent 场景里容易出问题

OpenClaw 的插件或 MCP 工具经常要对接各种异步系统。一个典型流程是：

1. Agent 调用工具 `start_analysis_task`，得到一个 `task_id`。
2. Agent 需要等待任务完成，再去获取结果。
3. 如果等待机制不可靠，Agent 要么过早返回“失败”，要么卡死占用上下文。

这个等待机制不只是一个简单的 sleep，它涉及超时、重试、状态转换、资源释放。如果直接用轮询，可能写出 1 秒一次的请求风暴；如果草率上推送，又会遇到连接断开、事件丢失、鉴权过期等一堆问题。

## 问题：轮询和推送的边界在哪里

轮询的优势是简单、无状态、容易排查。缺点是延迟由间隔决定，且请求量会随调用方数量线性增长。推送的优势是近实时、服务端压力小，缺点是需要维护长连接或公网回调，处理重连、幂等、鉴权，复杂度高。

所以取舍的核心不是“哪个技术更先进”，而是“你的场景能容忍多长的延迟，以及你愿意承担多少运维复杂度”。

## 做法：先量化，再选型，最后封装

我们可以在插件里实现一个简单的 `HeartbeatClient`，把轮询和推送都封装成同一种接口，业务层只调用 `wait_for_update(task_id, timeout)`。

### 第一步：量化三个参数

- 状态变化频率：是秒级、分钟级还是小时级？
- 可接受延迟：Agent 等 3 秒和等 30 秒，体验完全不同。
- 调用方数量：是单个本地进程，还是多个实例同时监听？

一个经验规则：如果延迟容忍大于 5-10 秒，且调用方少，优先用轮询；如果延迟要求小于 1 秒，或者调用方很多，再考虑推送。

### 第二步：轮询实现要点

轮询不是简单地 `while True: sleep(2)`。需要处理：

- **超时**：设置最大等待时间，超过后返回明确错误。
- **间隔自适应**：根据任务类型动态调整。例如，任务平均 30 秒完成，前 20 秒可以每 5 秒查一次，之后每 2 秒查一次。
- **抖动（jitter）**：避免多个客户端同时发起请求造成惊群，给间隔加一个随机偏移。
- **幂等**：状态接口最好返回带版本号或游标的数据，确保客户端不会重复处理同一条状态。

下面是一个简化版的 Python 伪代码：

```python
async def poll_status(task_id, timeout=60, base_interval=2.0):
    start = time.time()
    while time.time() - start < timeout:
        status = await client.get(f"/tasks/{task_id}/status")
        if status["state"] in ("completed", "failed"):
            return status
        # 自适应间隔：越接近超时，查得越勤
        elapsed = time.time() - start
        interval = max(0.5, base_interval * (1 - elapsed / timeout))
        await asyncio.sleep(interval + random.uniform(0, 0.3))
    raise TimeoutError(f"task {task_id} not finished in {timeout}s")
```

### 第三步：推送实现要点

如果决定用推送，优先选择 SSE（Server-Sent Events）而不是 WebSocket，因为 SSE 是单向的、基于 HTTP，更容易穿透代理，也更容易处理重连。WebSocket 适合需要双向通信的场景，但对 Agent 等待状态来说，通常只需要服务端推送。

推送链路必须处理三件事：

- **应用层心跳**：不要依赖 TCP keepalive，自己在协议层发 ping/pong，超时主动断开重连。
- **重连退避**：断线后不要立刻无限重试，使用指数退避 + 抖动，例如 1s、2s、4s、8s……上限 30s。
- **事件补拉**：连接断开期间可能丢事件，推送协议要带序号或游标，重连后先补拉再继续接收。

## 踩坑点：轮询和推送各自容易翻车的地方

**轮询的坑：**

- 固定 1 秒轮询，导致服务端 QPS 暴涨，尤其是多个 Agent 同时跑。
- 没有超时检查，任务已经失败，但客户端还在傻等。
- 状态接口返回完整列表，客户端重复处理已处理过的条目。

**推送的坑：**

- 用 WebSocket 但没做应用层心跳，中间设备空闲断开后客户端毫无感知。
- 重连逻辑没有退避，服务端刚恢复就被重连请求打挂。
- 推送事件丢失，但业务层没有补拉机制，导致状态永久不一致。

## 可复用建议

在 OpenClaw 插件里，建议把 heartbeat 做成一个独立模块，对外暴露统一接口，内部可以切换轮询和推送。这样前期用轮询快速跑通，后期如果延迟要求变高，再无缝切到推送。

- 状态查询接口设计成幂等，带 `version` 或 `cursor` 字段。
- 轮询间隔加 jitter，避免惊群。
- 推送优先用 SSE，重连用指数退避。
- 监控三个指标：轮询 QPS、推送重连次数、端到端延迟。
- 如果任务时间很长，暴露 `cancel` 接口，不要让 Agent 无限等下去。

## 总结

AI 助手的 heartbeat 设计，不是轮询和推送的二选一，而是根据延迟容忍度和运维复杂度做权衡。大部分本地插件和低频异步任务，轮询足够；只有真正需要近实时、多方协同的场景，才值得上推送。先算清楚账，再写代码，往往比直接引入一套复杂的推送框架更省心。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/0c575efb39eeec90.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/026778a366b38f5a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/92ae0764d3239725.png)

