---
title: AI 助手的 heartbeat 设计：轮询还是推送，先分清“活着”和“完成”
feedId: 34364
source: 综合讨论
publishedAt: 2026-08-23
---

## 背景

在 OpenClaw 的 Agent、MCP server 或插件里，经常会遇到同一个问题：AI 助手发起一个长任务后，怎么知道它是否还活着、执行到哪一步了？比如一个自动化流程要跑 3 分钟，宿主侧如果只等最终返回，一旦中间卡死，用户只能干等直到超时。于是我们做 heartbeat。

heartbeat 的设计通常被简化成“加个轮询”或“上个 WebSocket”。但实际落地时，真正花时间的不是选协议，而是先定义清楚心跳里到底装什么。下面按我们实践过的场景展开。

## 问题

轮询的优势是简单、无状态，客户端定时拉一次状态。代价是延迟至少一个间隔，并且如果多个 Agent 同时高频轮询，CPU、连接数和日志量都会上来。

推送的优势是及时，适合等待外部回调、任务完成事件。但它要维持长连接，处理断线重连、背压、重复消费；如果只是每分钟看一次进度，推送就是过度设计。

更大的问题是语义混在一起。健康心跳（liveness）、进度心跳（progress）、完成事件（result）经常用同一个字段，导致“进度没变”被误判成“进程死了”，或者“健康检查正常”掩盖了“业务已经失败”。

## 做法/步骤

第一步：定义事件类型。建议至少分四类：

- `liveness` / `health`：进程或 worker 是否存活
- `progress`：任务进度，带 `current`、`total`、`message`
- `result`：最终成功结果
- `error`：失败原因

第二步：选择通道。本地子进程优先用 stderr 或 Unix socket，不要占用 stdout。远程优先考虑 SSE 或 WebSocket；如果接收端在公网且不稳定，webhook 比长连接更务实。MCP server 场景要特别注意：stdio 是 JSON-RPC 通道，心跳和日志不要往 stdout 写，应该用 stderr 或 MCP 的 notification/logging。

第三步：做一个最小 heartbeat 模块。下面是一个缩减版：

```ts
type Heartbeat = {
  seq: number;
  kind: "liveness" | "progress" | "result" | "error";
  taskId: string;
  progress?: number;
  message?: string;
  ts: number;
};

class HeartbeatReporter {
  private seq = 0;
  constructor(private send: (hb: Heartbeat) => void) {}

  report(kind: Heartbeat["kind"], extra?: Partial<Heartbeat>) {
    this.send({
      seq: ++this.seq,
      kind,
      taskId: "task_001",
      ts: Date.now(),
      ...extra
    });
  }
}
```

第四步：选择轮询还是推送。我们的默认策略是：任务启动后先 5 秒轮询一次状态；进入关键等待阶段（例如等浏览器自动化结束、等外部 API 回调）再开启 SSE/WebSocket 推送；如果推送通道失败，降级为 10 秒轮询。这样兼顾实现成本和及时性。

在 OpenClaw 插件里，可以把 heartbeat 做成独立 service，让工具调用只负责触发，不直接散落发送心跳。宿主侧根据 `seq` 去重，避免重复消费。

## 踩坑点

1. **MCP stdio 污染**：有人在 MCP server 里用 `console.log` 输出心跳，宿主解析 JSON-RPC 直接失败。正确做法是输出到 stderr，或使用协议内 `notifications`。
2. **推送断线丢状态**：WebSocket 断开重连后，如果只从当前时刻开始推，中间进度会丢。需要客户端带上 `last_seq`，服务端按需补发或直接推一次全量 snapshot。
3. **高频率进度推送造成背压**：100 个任务每秒推一次，宿主处理不过来。对 `progress` 做节流，例如 500ms 或 1s 合并一次；`result`/`error` 必须即时。
4. **健康与进度耦合**：不要用“进度多久没更新”判断“服务是否活着”。可以单独发轻量 `liveness`，间隔 10-15 秒；任务超时单独设置。
5. **推送不可达时没有降级**：如果 webhook 没收到或 WebSocket 建连失败，不能一直等，必须有轮询兜底。

## 可复用建议

- 默认轮询，关键路径推送；不要一上来上长连接。
- 所有 heartbeat 带 `seq` 和时间戳，消费端做幂等。
- 心跳间隔与任务超时解耦，避免互相干扰。
- 在 OpenClaw/Agent 插件中，把 heartbeat 独立成模块，配置化间隔和通道。
- 监控 heartbeat gap：如果超过 2-3 个周期没收到，再触发告警，而不是单点判断。

## 总结

heartbeat 设计的核心不是轮询和推送哪个更先进，而是先分出“活着”“在推进”“已完成/失败”三种状态，然后选择最匹配的通道。多数自动化任务用轮询就够；只有需要及时取消、快速感知完成或等待外部回调时，才引入推送。工程上保留降级路径，比追求实时性更重要。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/110a5912a8c788bb.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/ba72423b4636a3b9.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/63eeacf51db01c2d.png)

