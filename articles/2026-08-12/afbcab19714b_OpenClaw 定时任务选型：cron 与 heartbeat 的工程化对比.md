---
title: OpenClaw 定时任务选型：cron 与 heartbeat 的工程化对比
feedId: 32809
source: 综合讨论
publishedAt: 2026-08-12
---

# OpenClaw 定时任务选型：cron 与 heartbeat 的工程化对比

## 背景：OpenClaw 中的两种触发原语

OpenClaw 作为编排多 Agent 的运行时，内置了两种周期性触发机制，分别对应不同的工作模型：

- **cron**：基于挂钟时间的调度，使用标准的 cron 表达式配置。当系统时间匹配表达式时触发任务，适合“每5分钟执行一次”或“每天9点执行”这类计划性工作。
- **heartbeat**：系统级心跳，从 OpenClaw 启动后按固定间隔（例如每10秒）触发回调，不关心具体是几点几分。通常用于轻量级巡检、保活、状态上报。

不少开发者在接入 MCP 工具或自动化流水线时，会混淆两者的适用场景。这篇笔记来自我们在维护一个多 Agent 数据管道时的实际踩坑，重点讨论什么时候该用 cron，什么时候该用 heartbeat，以及如何在 OpenClaw 中安全落地。

## 问题：看似可互换，实则会误用

最典型的误用模式有两种：

1. **用 heartbeat 轮询外部 API 做数据同步**  
   如果每30秒拉取一次第三方接口，并将结果写入数据库。这个逻辑如果写在 heartbeat 回调里，初看似乎正常，但真正上线后你会发现：  
   - 当 Agent 执行耗时工具调用时，heartbeat 会被推迟（取决于事件循环状态），间隔变得不稳定。  
   - 无法暂停或取消这个“假定时任务”，也难以记录执行历史。  
   - 由于 heartbeat 运行在 Agent 主循环上下文中，网络超时会直接阻塞整体响应。

2. **用 cron 做高频健康检查**  
   例如每5秒 ping 一个内部服务，记录响应时间。cron 的最小粒度一般是1分钟，更短间隔会引发大量调度开销，并且在任务执行时间超过间隔时，会产生任务堆积，需要复杂的并发控制。

本质问题在于：**任务是否有计划的、强时间属性的特征，还是只是系统生命期内需要的定期轻量动作**。选错原语会引入不必要的复杂度。

## 做法：两个典型场景的工程落地

### 场景一：用 cron 实现数据库定时同步

假设你需要每10分钟从外部数据源拉取增量记录，合并到本地存储，并且要求在系统重启后任务依然自动运行。

**配置 cron 任务：**

```yaml
# openclaw.yaml
cron:
  - name: data-sync
    schedule: "*/10 * * * *"
    job:
      type: agent_task
      agent_id: sync-agent
      input:
        operation: incremental_sync
```

在 `sync-agent` 的实现里，需要确保幂等性，通过外部存储记录上次成功的时间戳或游标。关键点：OpenClaw 的 cron 机制不支持自动去重，若上一次任务未完成且新触发点到来，默认会再启一个新实例。因此我们通过一个分布式锁（例如 Redis setnx）保证单实例运行：

```python
lock_key = "cron:data-sync:lock"
if redis.set(lock_key, "1", nx=True, ex=600):
    try:
        run_sync()
    finally:
        redis.delete(lock_key)
```

如果需要在 Agent 触发过程中使用 MCP 工具读取大文件，记得将任务拆分为子步骤，避免长时间占用 Agent 线程。

### 场景二：用 heartbeat 维持 WebSocket 连接

我们在工具层封装了一个基于 WebSocket 的实时数据源，需要自动重连并上报心跳。这里适合用 OpenClaw 的 heartbeat 钩子，每15秒检查连接状态，如果断开则触发重连，但不做重量级的数据拉取。

**注册 heartbeat 回调：**

```python
from openclaw import get_runtime

runtime = get_runtime()

@runtime.on_heartbeat(interval=15)
async def keep_ws_alive():
    if not ws_client.connected:
        await ws_client.reconnect()
    await ws_client.ping()
```

设计上必须保证这个回调的非阻塞性：重连里面设置较短超时（如3秒），失败后立刻返回，下一个心跳再重试，防止拖慢 Agent 思考循环。可以将状态上报到 OpenClaw 的内置 metrics 供 Dashboard 查看，但不要在回调中等待复杂 Agent 任务完成。

## 踩坑点

- **cron 与系统时间强绑定**  
  如果服务器发生 NTP 大幅度校时，可能导致某次触发提前或延后，甚至漏触发。对于不容忍丢失的任务，需要配合补偿机制（如记录 last_run 时间并在启动时 scan 一次）。

- **heartbeat 间隔漂移**  
  heartbeat 的触发时机并非绝对精确，它运行在事件循环中，如果 Agent 正在处理长对话或工具调用，间隔会变长。如果你的业务要求心跳间隔不大于某个阈值，要评估最大延迟是否能接受，不能拿 heartbeat 当 RTOS 用。

- **heartbeat 内不能使用阻塞 I/O**  
  尤其是同步的 HTTP 请求或数据库查询，极易导致整个 Agent 运行时卡顿。务必使用异步客户端，且设置合理的超时。

- **任务堆积与静默丢失**  
  cron 的重复执行没有内置的 finished check，必须在业务层处理。如果某次任务执行时间超过了间隔，新实例会被创建，可能引起资源竞争或数据重复。建议在生产环境强制使用分布式锁或任务队列。

## 可复用建议

一个快速判断的工程化 checklist：

| 特征 | 推荐选择 | 原因 |
|------|----------|------|
| 有明确时间表（每5分钟、每天9点） | cron | 语义清晰，可持久化，可观测 |
| 任务较重、允许失败重试 | cron | 方便做状态管理和补偿 |
| 需要暂停/启用/查看历史 | cron | OpenClaw 可扩展 JobStore |
| 纯状态检查、保活、心跳上报 | heartbeat | 开销小，随系统生命周期自动启停 |
| 需要极轻量且频率较高（<10秒） | heartbeat | 避免调度器压力 |
| 与 Agent 内部事件紧密耦合 | heartbeat | 能够访问运行时上下文 |

落地时，可以封装一个工厂函数来创建标准的 cron 任务或 heartbeat 回调，强制接入幂等锁、异常捕获和 metrics 打点，降低每处重复实现的成本。在 MCP 工具侧，凡涉及外部长连接 keepalive，都优先考虑 heartbeat，并把重连逻辑内聚在工具内部，而不是靠 Agent 循环手动维持。

## 总结

cron 和 heartbeat 在 OpenClaw 中并不是互相替代，而是对应两种任务属性：**计划性作业**与**周期巡检**。前者需要时间表、并发控制和可恢复性；后者需要轻量、非阻塞和与运行时生命周期绑定。选中正确的原语，可以避免把简单的健康检查做成复杂的任务调度系统，也能防止重要的批处理任务被塞进不稳定的心跳循环里。在工程化落地时，为每种类型建立统一模板，接入锁与可观测性，就可以让定时任务真正可靠地运行在 Agent 主干旁边。

---

