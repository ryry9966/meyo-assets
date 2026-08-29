---
title: AI 助手的 heartbeat 设计：轮询 vs 推送的工程化取舍
feedId: 35285
source: 综合讨论
publishedAt: 2026-08-30
---

## 背景

在 OpenClaw 生态里，AI 助手经常要跨进程、插件、MCP 工具和自动化任务协同。一个典型场景是：Agent 调度一个异步工具后，需要知道这个工具是否还在跑、进度如何、结果是否就绪。heartbeat 不只是“心跳保活”，还承担状态同步、断线检测、任务取消等职责。

但很多实现一上来就纠结“轮询好还是推送好”，最后变成过度设计。真正影响稳定性的，往往是代理超时、重连风暴、多实例重复触发这些边角问题。

## 问题

轮询简单直接：客户端每隔 N 秒拉一次 `/status`，实现成本低，但延迟受间隔约束，空轮询会浪费资源。推送通常用 SSE 或 WebSocket：服务端有变化就下发，实时性好，但连接管理更复杂，需要处理静默断开、代理超时、客户端状态同步。

取舍的核心不是“哪个技术更先进”，而是你的 heartbeat 到底在同步什么：

- 存活探测：低频即可，轮询足够。
- 进度/日志流：实时性要求高，适合 SSE。
- 双向信令：比如取消任务、交互式调试，才需要 WebSocket。

## 做法/步骤

1. **先明确语义**  
   把“存活探测”和“进度流”分开。存活探测可以低频轮询，进度流按需订阅，不要把两者耦合在一个接口里。

2. **短任务先做轮询 MVP**  
   间隔 2–5 秒，超时 3 秒，加随机抖动避免惊群。接口保持幂等，返回 `sequence` 字段，客户端可以据此去重。

3. **实时性场景切 SSE**  
   模型 token 输出、日志尾部、任务进度等单向流场景优先 SSE。服务端返回 `text/event-stream`，每 15–30 秒发送注释帧 `: keep-alive` 或显式 heartbeat 事件，防止中间设备关闭空闲连接。

4. **WebSocket 只留给双向信令**  
   心跳帧可以用协议层 `ping/pong`，也可以用应用层 `{"type":"heartbeat"}`。但不要为了一个状态查询就上 WebSocket。

5. **客户端做状态机**  
   至少包含 `disconnected -> connecting -> idle -> active -> stale -> reconnecting`。用 `last_seen` 判断 stale，超过 `2 * heartbeat_interval + rtt` 就认为连接不健康。

6. **重连退避要克制**  
   指数退避加抖动，上限 30–60 秒。例如：`delay = min(cap, base * 2^retry) + random_jitter`，避免重连风暴。

7. **可观测**  
   记录心跳成功率、往返时延、消息积压、重连次数。这些指标比协议选择更能反映真实健康度。

一个可参考的配置片段：

```yaml
heartbeat:
  mode: sse
  interval: 20s
  timeout: 5s
  stale_after: 45s
  reconnect:
    base: 1s
    cap: 30s
    jitter: 0.3
```

## 踩坑点

- **代理超时**  
  Nginx 默认 `proxy_read_timeout 60s`，SSE/WebSocket 长连接会被静默切断。需要为 `/stream` 路径单独调大，或关闭缓冲。

- **HTTP/1.1 连接数限制**  
  浏览器同域名并发连接通常只有 6 条，多标签页容易占满。SSE 场景尽量用 HTTP/2，或提前提醒用户。

- **移动端/休眠**  
  手机锁屏后定时器可能被冻结，轮询直接失效；推送长连接也可能被系统回收。需要监听 `visibilitychange`，回到前台立即补一次状态同步。

- **多实例重复触发**  
  如果 heartbeat 不仅读状态，还驱动任务调度，多实例同时检查同一个任务会重复执行。heartbeat 本身应设计为无副作用的状态读取；需要触发动作时，用分布式锁或队列单消费者。

- **心跳回调里做重业务**  
  每次 heartbeat 都扫全量任务会导致 CPU 尖刺。心跳回调只更新时间戳、检测超时；重活放到独立 worker。

- **消息乱序/重复**  
  推送通道不保证顺序，必须用业务侧 `sequence` 或时间戳去重，否则可能重复触发任务或写入脏数据。

## 可复用建议

封装一个薄薄的 `HeartbeatClient`，对外只暴露 `onStateChange`、`onStale`、`forceCheck` 几个接口。把 interval、timeout、retry 放进配置中心，不同环境可以独立调参。

默认轮询，按需升级推送。不要一上来就 WebSocket。监控三个核心指标：心跳成功率、最后心跳时间、stale 次数。这三个指标基本能覆盖 90% 的线上问题。

## 总结

轮询和推送不是互斥的技术路线，而是同一状态同步需求的两级策略。工程上先保证简单可控的轮询，再用 SSE 补实时性，WebSocket 留给双向信令。真正的成本不在协议选择，而在重连、去重、代理超时和多实例这些边角问题。把 heartbeat 做成一个可观测的薄组件，比纠结轮询还是推送更重要。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/14aca2aababffa52.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/2cdbe5266a5caa34.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/6033a8b9b3f66487.png)

