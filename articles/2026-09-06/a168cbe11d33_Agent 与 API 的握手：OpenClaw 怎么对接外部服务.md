---
title: Agent 与 API 的握手：OpenClaw 怎么对接外部服务
feedId: 36331
source: 综合讨论
publishedAt: 2026-09-06
---

## 背景

OpenClaw 这类常驻 Agent 的价值，很大程度取决于它能"够到"多少真实服务：查库存、发工单、拉报表。模型本身不联网，真正干活的是工具层。所以对接外部 API，几乎是每个部署者绕不开的活。

## 问题

对接不是发个 HTTP 请求那么简单。人调 API 时，鉴权、超时、重试、字段裁剪都是我们写死的逻辑；换成 Agent 之后，这些决策变成了"模型现场判断"。于是问题变成：怎么把一段一次性的 HTTP 调用，沉淀成 Agent 手里可靠、安全、可重复使用的能力。

## 做法

我目前的路线是把第三方 API 包成薄薄的 MCP server，再挂进 OpenClaw。现成的社区 server 能用就用，没有的再自己包。以一个只读天气接口为例：

```ts
// 只做鉴权、超时、错误翻译、字段裁剪
server.tool("get_weather", "查询城市未来三天天气", { city: z.string() },
  async ({ city }) => {
    const res = await fetch(url, {
      headers: { Authorization: `Bearer ${process.env.WEATHER_TOKEN}` },
      signal: AbortSignal.timeout(8000),
    });
    if (!res.ok) return err(`upstream ${res.status}`);
    return ok(pick(await res.json(), ["date", "temp", "desc"]));
  });
```

然后在配置文件（openclaw.json，不同版本字段名略有差异）的 mcpServers 里注册，token 走 env 注入：

```json
"weather": { "command": "node", "args": ["server.ts"],
  "env": { "WEATHER_TOKEN": "sk-..." } }
```

顺序上建议：先用只读接口打通注册 → 发现 → 调用的链路，确认参数 schema 和返回格式稳定，再考虑包写操作。每个上游服务一个独立 server，别塞成大杂烩。

## 踩坑点

- **工具描述含糊**：模型会误调用或干脆不调。一个工具只做一件事，参数用 JSON Schema 写清单位和枚举值。
- **没有超时和错误翻译**：上游 5xx 时只抛异常，Agent 循环会卡住或盲目重试。超时 + 有限重试，并把失败翻译成一句话返回，让它知道该放弃还是改参数。
- **写接口不幂等**：Agent 重试一次就是重复下单。写操作要么带幂等键，要么先 dry-run，要么强制人工确认。
- **返回不裁剪**：上游 JSON 整包塞回上下文，token 烧得快还干扰决策，只留 Agent 真正需要的字段。
- **密钥进上下文**：token 一旦出现在系统提示词或聊天记录里就收不回来了，只走 env，别让模型"看见"它。

## 可复用建议

- 薄封装原则：鉴权、超时、裁剪、限流全放 wrapper 层，Agent 只面对干净的输入输出。
- 权限最小化：只读 token 优先，能加 IP 白名单就加。
- 每次工具调用记结构化日志（耗时、状态码、参数摘要），排障全靠它。
- 把 schema 当合同维护：上游改字段时，先想到 Agent 的调用可能跟不上。

## 总结

Agent 与 API 的握手，本质是把人调 API 的那套工程纪律平移给模型：schema 是合同，超时和错误翻译是容错，最小权限是底线。OpenClaw 的 MCP 机制已经把"注册与发现"做得很薄，剩下的质量取决于每个 wrapper 写得是否克制。先只读后写入，先一个服务再一批服务，链路稳了再谈自动化。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/abc76132b1508015.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/85be7429224f3a2e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/1df91776b8b744c2.png)

