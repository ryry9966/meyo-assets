---
title: Agent 与 API 的握手：OpenClaw 对接外部服务的工程化实践
feedId: 35021
source: 综合讨论
publishedAt: 2026-08-28
---

## 背景：Agent 的能力边界由外部服务决定

OpenClaw 里跑起来的 Agent，如果只靠模型内部知识，很容易变成“聊天机器人”。真正能落地的场景，不管是查库存、下单、发通知还是拉取报表，都需要 Agent 稳定地调用外部 API。

但 API 是强契约，模型输出是非确定性的。两者握手时，常见的问题不是“能不能调通”，而是调通之后如何保持稳定、安全、可排障。这篇文章不讨论某个具体供应商，只讲工程上怎么把外部 HTTP 服务接到 OpenClaw Agent 里。

## 问题：不是 requests.get 那么简单

直接在一个工具函数里写：

```python
r = requests.get("https://api.example.com/orders", headers={"Authorization": "Bearer xxx"})
return r.json()
```

这种做法在 demo 里能跑，上生产会很快暴露问题：

- 模型传参不稳定，类型、缺省、多余字段都会发生；
- API 超时、限流、5xx 没有处理策略；
- 响应 JSON 太大，直接把几万 token 塞进上下文；
- 鉴权信息容易被日志或错误信息带出去；
- 流式接口、分页接口没有边界控制。

所以对接外部服务，核心是给“模型的不确定性”套一层“工程确定性”。

## 做法：把外部服务封装成可校验的工具

### 1. 每个外部服务一个工具，定义输入输出 Schema

在 OpenClaw 中注册工具时，不要只给一个 `**kwargs` 就完事。推荐用 Pydantic 模型定义输入：

```python
from pydantic import BaseModel, Field

class GetOrderInput(BaseModel):
    order_id: str = Field(description="订单号")
    include_items: bool = Field(default=False, description="是否包含明细")
```

输出也尽量裁剪。API 返回 40 个字段，模型决策可能只需要 `status`、`total_amount`、`updated_at` 三个。

### 2. 统一 API client，配置与鉴权分离

不要在工具函数里硬编码 host 和 key。建议维护一个 `api_client` 模块，所有外部服务走同一个 session，统一设置超时、重试、headers。

```python
client = httpx.AsyncClient(
    base_url=os.environ["ORDER_API_BASE"],
    timeout=httpx.Timeout(10.0, read=20.0),
    headers={"Authorization": f"Bearer {os.environ['ORDER_API_TOKEN']}"}
)
```

这样改环境、轮换密钥、调整超时不用改工具逻辑。

### 3. 给模型写清楚触发条件与副作用

工具描述是模型决定是否调用的唯一依据。描述里至少包含三件事：

- 什么时候应该调用；
- 每个参数的业务含义；
- 调用后会不会产生副作用，比如创建订单、发送短信。

只读接口可以自动重试；有副作用的接口，描述里要写清楚“成功后会改变业务状态”，必要时让模型先确认。

### 4. 错误映射，不要把堆栈抛给模型

外部 API 返回 401、429、500 时，模型不需要看到 HTML 错误页或长堆栈。捕获后转成结构化文本：

```text
[get_order] 调用失败：上游返回 429，已重试 2 次，建议稍后重试。
```

这比抛异常让 Agent 崩溃好得多，也方便在 OpenClaw 的 trace 里定位。

### 5. 本地先 mock 再连真服务

联调时用一个简单 mock server 返回固定 JSON。先把 Schema、超时、错误路径验证完，再连真实 API。真实 API 的限流和不稳定不会干扰你的工具逻辑调试。

## 踩坑点

**参数漂移**  
模型有时会传 `None`、字符串数字、多余字段。用 Pydantic 校验只是第一步，校验失败后要把错误信息友好地返回给模型，让它重新组织参数，而不是直接终止。

**重试策略不一致**  
GET 类幂等请求可以自动重试 2 次；POST、PUT、DELETE 不要自动重试，否则可能重复创建订单。重试只对 429、502、503 等状态码生效，401 重试没有意义。

**鉴权信息泄露**  
日志里不要打印完整 headers。错误信息里如果包含上游返回的请求头，要脱敏。密钥只从环境变量读取，不写进工具描述或 prompt。

**流式响应解析**  
SSE 或 NDJSON 接口逐行处理时，注意缓冲区可能把一行拆成两半。要按行边界解析，遇到 `data: [DONE]` 及时退出。

**分页无限循环**  
外部接口如果有分页，不要在工具里 `while True` 翻页。设置 `max_pages` 上限，默认只取前 1-2 页，超出后返回 `has_more: true` 让 Agent 决定是否继续。

**大响应吃光上下文**  
这是最容易被忽视的性能问题。一个订单详情可能带出几万 token 的商品快照。工具层就应该裁剪字段，而不是让模型自己从大 JSON 里找答案。

## 可复用建议

- **封装一个 `create_api_tool` 工厂函数**：统一处理异常捕获、超时、重试、日志脱敏、Schema 校验。每个具体工具只传接口定义和响应裁剪逻辑。
- **用 MCP 时把认证放在 server 端**：如果你用 MCP 接入外部服务，建议把 HTTP 细节封装在 MCP server 内部，OpenClaw 作为 MCP client 只看到标准化工具。认证、重试、限流都在 server 层解决。
- **做防腐层**：不要在 Agent 工具里直接写死供应商 API 细节。第三方 API 一旦升级版本，你的工具层应该能独立替换，不影响 Agent 的决策逻辑。
- **记录可观测指标**：每次外部调用至少记录耗时、状态码、重试次数、是否裁剪字段。排查 Agent 行为异常时，外部服务调用日志会是最重要的线索。

## 总结

OpenClaw 对接外部服务，本质上是把模型的不确定输出，约束到外部 API 的确定性契约上。做法并不复杂：定义 Schema、统一 client、错误映射、响应裁剪、谨慎重试。真正拉开差距的，是这些细节能不能在工程上稳定复用。

一次成功的“握手”，不是让 Agent 成功调用一次 API，而是当 API 超时、限流、参数漂移时，Agent 依然能给出可预期的行为。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/ac78a354b6a2110b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/2f740016a5d5664f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/453eb5765c6a9e5c.png)

