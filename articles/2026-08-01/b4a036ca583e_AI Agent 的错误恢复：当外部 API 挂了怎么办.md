---
title: AI Agent 的错误恢复：当外部 API 挂了怎么办
feedId: 31142
source: 综合讨论
publishedAt: 2026-08-01
---

## 背景：Agent 比想象中更脆弱

在 OpenClaw 这类可扩展的 Agent 框架里，我们习惯了通过插件或 MCP 服务给模型“接上手臂”——调用天气 API、搜索接口、数据库查询、发送消息。Agent 的行为链路因此变得强大，但也引入了一个老问题：**外部依赖随时会失败**。

一次超时、一个限流 429、一次上游服务重启，如果没有处理，整条 Agent 任务链就会直接抛异常，前端用户看到的可能是一句冷冰冰的“工具调用失败”，甚至什么也不返回。对于已经跑了一半的自动化流程（比如定时生成报告、自动处理工单），这种“硬失败”代价很高。

这次我们聚焦一个具体的场景：**你为 OpenClaw 写了一个 MCP 工具，底层调用第三方 REST API，当这个 API 挂了，你如何让 Agent 还能体面地完成任务，而不是直接报错**。

---

## 问题拆解：API 崩溃的几种面孔

我们从工程角度把外部 API 的失败分成几类，因为不同错误需要的恢复策略完全不同：

1. **瞬时网络故障**（连接超时、DNS 解析偶发失败、`ECONNRESET`）  
2. **服务端临时过载**（HTTP 503、504、502）  
3. **限流**（HTTP 429，带有 `Retry-After` 头）  
4. **客户端错误**（401 鉴权过期、403 无权、400 参数错误）—— 这类通常不能重试，需要人工介入或修复配置  
5. **返回异常数据**（状态码 200 但 body 是错误页面 HTML，或 JSON 结构突变）

前三种适合**自动重试**，第四种应当**快速失败并告警**，第五种需要**数据校验 + 降级兜底**。

---

## 做法与步骤：从重试到优雅降级

### 1. 封装一个带策略的 HTTP 客户端

不要在每个 MCP 工具里裸写 `fetch` 或 `axios`。建议抽离一个 `apiClient`，集中处理错误分类、重试、熔断。伪代码层级如下：

```ts
async function callWithRecovery(url, options) {
  const maxRetries = 3;
  const backoff = [1000, 2000, 4000]; // 指数退避，实际加 10%~30% jitter
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      const resp = await fetch(url, { timeout: 8000, ...options });
      // 限流时读 Retry-After
      if (resp.status === 429) {
        const retryAfter = resp.headers.get("Retry-After") || backoff[attempt] / 1000;
        await sleep(retryAfter * 1000);
        continue;
      }
      // 5xx 重试
      if (resp.status >= 500) throw new Error("Server Error");
      // 2xx 仍需校验 body
      const data = await resp.json();
      if (!isValidData(data)) throw new Error("Invalid data format");
      return data;
    } catch (err) {
      if (attempt === maxRetries) throw err;
      if (isRetryable(err)) {
        await sleep(backoff[attempt] + Math.random() * 500);
      } else {
        throw err; // 不可重试的错误直接抛出
      }
    }
  }
}
```

### 2. 为 MCP 工具增加降级返回

在 MCP 工具的实现里，可以用 `try/catch` 包裹上述客户端。如果最终还是失败，**不要抛出未捕获异常让 MCP 服务器崩溃**，而是返回一个标准化的降级结构：

```json
{
  "error": "temporary_unavailable",
  "message": "天气数据暂不可用，以下为 30 分钟前的缓存结果",
  "cached_at": "2025-01-01T10:00:00Z",
  "data": { ... }
}
```

OpenClaw 在拿到这个结构化消息后，可以在 prompt 里让模型自然地告知用户：“当前无法获取最新数据，我展示的是稍早前的信息，请注意时效性。” 这样体验远优于一个无响应的死界面。

### 3. 熔断器：不要反复撞南墙

如果上游 API 已经持续故障 30 秒，何必每个工具调用都去重试 3 次？引入一个轻量熔断器：连续失败 N 次（如 5 次），在接下来一段时间内（如 60 秒）直接走降级逻辑，甚至不再发起请求。关键状态可以放在内存或轻量 KV 中，MCP 服务器重启后重置也无妨，总比打崩自己强。

```ts
if (circuitBreaker.isOpen()) {
  return fallbackResponse("circuit_open");
}
```

### 4. 让 Agent 知道“我在用降级数据”

在 OpenClaw 的系统或工具描述里，可以追加一句指令：

> 当工具返回中包含 `error: "temporary_unavailable"` 或 `cached_at` 字段时，请在回复中明确告知用户当前使用了非实时数据，并说明可能存在的偏差。

这能显著提升最终回答的可信度，而不是假装一切正常。

---

## 踩坑点：这些细节会让你半夜爬起来修

- **重试风暴**：如果不设最大重试次数和超时，一个下游服务抖动可能把整个 Agent 的调用栈耗尽。务必同时设置**单次请求超时**和**总重试时间上限**。  
- **缓存数据过期仍被当作“正常”**：降级逻辑如果没有给返回数据打上鲜明的时间标记，模型可能会把 2 小时前的股价当作实时数据告诉用户，这是生产级事故。永远在降级 payload 里携带 `cached_at` 或 `stale_seconds`。  
- **熔断状态没被重置**：如果你的熔断器只是计数累加而从不清空，最终所有工具都会永久降级。建议用时间窗口计数，或基于连续失败数，成功后立即重置。  
- **业务代码与恢复逻辑耦合**：每个工具里散落大量 if/else 不仅难维护，还容易在新增工具时遗漏。将恢复能力下沉到 `apiClient` 或中间件，工具只关注正常业务逻辑。

---

## 可复用建议：给 OpenClaw 用户的实践清单

1. **抽象一个 `ReliableAPIClient`**，支持重试、熔断、数据校验、降级回调，所有 MCP 工具统一使用。  
2. **为你的 MCP 工具定义标准错误返回格式**，包含错误码、人类可读原因、降级数据、数据新鲜度。  
3. **在 Agent 的系统指令里写清楚降级行为**，让模型知道如何解释“使用了缓存”。  
4. **对关键路径做故障注入测试**：用工具 mock 或网络代理故意返回 500 / 超时，观察 Agent 是否按预期降级，日志是否清晰。  
5. **监控降级率**：如果某个 API 的降级率突然从 0.5% 跳到 15%，你的重试和熔断可能正在掩盖一个需要人工介入的真实故障。

---

## 总结

AI Agent 的可靠性不是建立在“所有依赖都永远可用”的假设上，而是**在依赖不可用时仍能给出合理解答**。对外部 API 的错误恢复不能只靠模型自己猜，工程侧需要提供重试、熔断和清晰的降级信号。把这些逻辑封装进可复用的调用层之后，你会发现自己从一个祈祷 API 别挂的开发者，变成了一个即使挂了也能轻松应对的开发者。OpenClaw 的灵活插件体系正好为此提供了理想的落地土壤，值得你在下一个 Agent 项目里就动手践行。

---

