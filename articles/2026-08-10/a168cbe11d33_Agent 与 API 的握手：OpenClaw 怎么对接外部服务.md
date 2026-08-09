---
title: Agent 与 API 的握手：OpenClaw 怎么对接外部服务
feedId: 32325
source: 综合讨论
publishedAt: 2026-08-10
---

## 背景

Agent 一旦进入真实生产环境，“只能在沙箱里推理”的状态就必须结束。查天气、读工单、调企业内部接口——这些操作最终都会落到 HTTP API 上。OpenClaw 提供了标准化的工具层，但把 Agent 稳定、安全地接上外部服务，仍然需要一套工程化的对接模式，而不是零散的胶水脚本。

## 问题

直接在 prompt 或自定义 Python 胶水里硬编码调用逻辑，会带来一连串工程债：

- API 密钥散落在仓库、配置和脚本里，轮换困难；
- 外部接口返回格式一变，Agent 的输出就不可控；
- 没有统一的重试、超时、降级机制，一个慢 API 就能拖死整个 Agent 推理；
- 调用逻辑和推理逻辑耦合，难以独立测试与灰度。

因此需要一种模式，让 Agent 像调用本地函数一样安全地触发外部服务，同时保留足够的可观测性与复用性。

## 做法：OpenClaw 的两条对接路径

### 1. 直接 HTTP 工具调用

OpenClaw 允许在工具配置中直接声明 HTTP 类型的 handler，让 Agent 自主发起请求。以获取城市天气为例，可以在 `openclaw.yaml` 里定义：

```yaml
tools:
  - name: get_weather
    description: "获取指定城市的实时天气信息"
    parameters:
      type: object
      properties:
        city:
          type: string
          description: "城市名称，如 Beijing"
    handler:
      type: http
      http:
        method: GET
        url: "https://api.weatherapi.com/v1/current.json?key=$WEATHER_API_KEY&q=$city"
        timeout: 5000
        retry:
          max_retries: 2
          backoff: exponential
        response_mapping:
          temperature: "$.current.temp_c"
          condition: "$.current.condition.text"
```

关键点：
- 用环境变量传入密钥（`$WEATHER_API_KEY`），绝不写在配置文件里；
- 通过 `response_mapping` 只提取 Agent 关心的字段，避免把整个 JSON 灌进上下文；
- 显式控制超时和重试策略，防止连锁故障。

### 2. 通过 MCP 服务器代理

当外部服务逻辑复杂（例如需要签名算法、Token 管理、多步组装）时，直接将调用逻辑内嵌在 OpenClaw 的工具定义里会很难维护。更好的方式是使用 MCP（Model Context Protocol）服务器作为统一代理层。

部署一个独立的 MCP 服务，比如 `weather-mcp`，暴露标准化的工具接口，内部封装真实 API 的所有细节。OpenClaw 端只需在配置中加入该 MCP 服务器的引用：

```yaml
mcp_servers:
  weather:
    url: http://localhost:5001/mcp
    transport: http
```

Agent 侧自动获得 MCP 服务器暴露的工具列表，不再需要配置任何 HTTP 细节。这带来了几个好处：

- **关注点分离**：外部接口的认证、重试、缓存都在 MCP 服务里处理，Agent 只关心“调用天气工具”这件事；
- **独立迭代**：API 升级、换供应商时，只需改 MCP 服务代码，Agent 方无感知；
- **安全边界清晰**：密钥只存在于 MCP 服务的运行环境中，OpenClaw 容器完全接触不到。

## 踩坑点

### 环境变量未正确传递
本地测试时没问题，打包进 Docker 或推上 CI 后就频繁报错 `401 Unauthorized`，原因往往不是密钥错误，而是环境变量根本没注入到 OpenClaw 的进程中。务必将 API 密钥通过启动命令或编排文件显式传入，并在启动日志里打印非敏感部分以确认加载成功。

### 响应格式噪音
外部 API 返回的数据往往十分冗余，直接喂给 Agent 容易导致它提取出“营销文案”而非真实数据，甚至产生幻觉。解决方案：强制使用 `response_mapping` 或 MCP 端的瘦身层，只返回结构化的关键字段。

### 重试风暴
API 提供方偶尔会返回 429 或 503，若没有合理重试上限，Agent 会不断重试直到整个任务超时。在生产环境中必须限制总调用时长（`timeout` + 重试次数 × 退避时间）并设置熔断规则。

### Token 过期与刷新
对于需要 OAuth2 或短期 Token 的 API，Agent 本体无法完成刷新操作。应将刷新逻辑实现在 MCP 服务器内部，或使用专门的凭证 MCP 服务器，对 Agent 完全透明。

## 可复用建议

- **以 MCP 为边界封装外部系统**：每一个外部依赖对应一个 MCP 服务，Agent 对能力编程，不对具体 API 编程。
- **统一密钥管理**：所有密钥通过环境变量注入或 Secret 管理服务获取，不在工具描述或配置中硬编码。
- **契约校验**：对每个外部 API 的返回数据做 JSON Schema 校验，避免坏数据进入 Agent 上下文。
- **合理使用缓存**：对于变化频率低的数据（汇率、节假日等），在 MCP 层加短周期缓存，降低延迟和 API 成本。
- **实施熔断与降级**：连续失败时返回预设的安全兜底信息，避免 Agent 推理卡死。

## 总结

Agent 与外部 API 的握手，不应是一次性的脆弱对接，而应当成为一套可复用的工程范式。OpenClaw 提供了从直接 HTTP 工具到 MCP 代理的完整弹性，让团队可以根据复杂度灵活选择。把密钥、重试、缓存和格式处理下沉到 MCP 服务，让 Agent 专注于推理和决策，这样的架构才能经得起生产环境的长期考验。

---

