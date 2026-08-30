---
title: Agent 与 API 的握手：OpenClaw 对接外部服务，先把边界做清楚
feedId: 35360
source: 综合讨论
publishedAt: 2026-08-30
---

## 背景

在 OpenClaw 里，工具、插件、MCP 最终都会落到一种执行模型：Agent 产出结构化参数，执行单元访问外部服务，再把结果裁剪回模型。外部服务不是“接上就能用”的开关，而是一个需要明确边界的上游依赖。

## 问题

实际踩坑里，Agent 调外部服务最常见的问题不是协议不通，而是工程边界不清晰：不设超时导致工具卡死；429/5xx 直接判定失败；上游错误原文几百字节塞回上下文；Token 混进日志；大响应没有分页造成上下文污染；POST 重试导致重复创建资源。这些问题不会让 API 不可用，但会让 Agent 表现非常不稳定。

## 做法/步骤

1. 先收敛工具签名。不要一次性暴露整个 API 表面，只给 Agent 最小参数集。比如创建工单只放 `title`、`body`、`priority`，不要在工具里暴露内部字段。
2. 用环境变量维护 Base URL、Token、超时、重试次数。描述里只写行为，不写鉴权。
3. 封装统一 HTTP 访问层。每个外部服务一个工具，内部做超时、重试、错误归一化和响应裁剪。
4. 返回结果只保留对象用于后续步骤或观察的字段，如 `id`、`url`、`status`。
5. 错误要分类：4xx 作为业务错误原样返回；429 和 5xx 触发短重试；超时返回固定信息。

一个轻量处理示例如下：

```ts
export async function createExternalIssue(input: { title: string; body: string }) {
  const res = await fetch(`${env.EXT_BASE_URL}/issues`, {
    method: "POST",
    headers: {
      "Authorization": `Bearer ${env.EXT_TOKEN}`,
      "Content-Type": "application/json"
    },
    body: JSON.stringify(input),
    signal: AbortSignal.timeout(Number(env.EXT_TIMEOUT_MS) || 8000)
  });

  if (res.status === 429 || res.status >= 500) {
    throw new RetryableError(`upstream ${res.status}`);
  }

  if (!res.ok) {
    const text = await res.text();
    throw new Error(`upstream ${res.status}: ${text.slice(0, 300)}`);
  }

  const data = await res.json();
  return { id: data.id, url: data.url, status: data.status };
}
```

如果目标服务已有现成 MCP server 或 OpenAPI，可以先用 MCP 包一层；否则按上述方式做成 OpenClaw 的工具或插件。

## 踩坑点

- 不设超时是最常见事故。请求挂起会直接拖累整个 Agent 循环，务必使用 `AbortSignal.timeout` 或等价机制。
- 对非幂等写操作盲目重试。创建、支付、发送类接口要使用上游支持的 `Idempotency-Key`；不支持就不要自动重试，只对 429、连接失败等可判定场景重试。
- 把原始错误整段塞回给模型。错误里可能包含 HTML、堆栈或无关字段，要截断为可读摘要。
- 大列表不做分页。列表工具应设 `limit`、`offset` 或 `pageToken`，默认只取第一页，避免上下文爆炸。
- 在日志里输出完整请求头。脱敏 `Authorization`，记录 `requestId` 比记录 body 更有用。

## 可复用建议

- 一个工具只做一件事。参数少、描述准，Agent 选择工具的成功率会明显提升。
- 给写操作加 `dry_run` 参数。先让 Agent 或人在不产生副作用的前提下验证参数结构。
- 用 enum 约束参数。状态、优先级等字段不要用开放字符串。
- 录下真实响应做 fixture，离线跑工具 handler，别每次都打真实 API。
- 关注指标：P95 延迟、按状态码统计的错误率、超时率。外部服务劣化时，Agent 的行为会很快暴露出来。

## 总结

OpenClaw 对接外部服务，关键不是套一个 `fetch` 或塞一个 MCP，而是把工具当成稳定接口来维护。边界清楚、错误可读、副作用可控，Agent 和外部服务的握手才不会变成随机故障。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/4240c0e22c441e6e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/c1e58fea504d2334.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/540092417c9551f5.png)

