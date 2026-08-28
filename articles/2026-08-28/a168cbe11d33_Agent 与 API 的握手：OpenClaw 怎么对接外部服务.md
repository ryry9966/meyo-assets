---
title: Agent 与 API 的握手：OpenClaw 怎么对接外部服务
feedId: 35093
source: 综合讨论
publishedAt: 2026-08-28
---

## 背景：Agent 不会自己伸手

在 OpenClaw 里，模型负责决策，真正访问外部服务的是执行层。常见做法是把外部 API 封装成 Tool、MCP server 或插件，让模型只负责选择调用和填参数。很多集成问题不是模型能力不够，而是握手层做得太随意：schema 不一致、错误处理缺失、鉴权失控。

## 问题：别让模型直接“裸调” API

模型只会产生文本，不会真正发出 HTTP 请求。如果让模型生成 URL 或 curl 命令再由执行层执行，参数漂移、注入和不可控重试会很常见。更稳妥的方式是：外部服务暴露为显式工具，模型选择调用并填参，实际请求由受控代码完成。

常见的失败点：
- 工具描述和真实 API 不一致，模型乱填参数。
- 返回体过大，挤爆上下文。
- 外部 API 超时或 5xx，被模型复述成“已完成”。
- 写操作缺少幂等设计，重试造成重复数据。

## 做法/步骤

**1. 先划边界、放权限**

给外部服务单独建最小权限 token，只授权需要的 scope。密钥不要写进系统提示词、工具描述或代码仓库。

**2. 封装薄函数**

把 API 包装成参数显式、返回结构化 JSON 的函数。示例：

```python
def get_weather(city: str, date: str) -> dict:
    resp = requests.get(
        "https://api.example.com/weather",
        params={"city": city, "date": date},
        headers={"Authorization": f"Bearer {API_TOKEN}"},
        timeout=5,
    )
    resp.raise_for_status()
    data = resp.json()
    return {
        "city": data["city"],
        "date": data["date"],
        "condition": data["condition"],
        "temp_c": data["temp_c"],
    }
```

关键在于裁剪返回字段，不要直接返回原始 JSON。

**3. 在 OpenClaw 中注册为工具**

使用 OpenClaw 的 Tool / MCP 配置把函数暴露给 Agent。如果走 MCP，就在 MCP server 的 `list_tools` 里声明；如果走本地插件，就在工具注册表里声明输入 schema。工具描述写清参数类型、格式和副作用，例如“date 使用 YYYY-MM-DD”。

**4. 设置超时、重试与错误映射**

HTTP 客户端设超时；对网络错误和 5xx 做有限重试，4xx 不重试。异常映射成稳定消息，例如 `weather_unavailable: upstream timeout`，避免堆栈进入模型上下文。

**5. 加审计和干跑模式**

记录 tool call 参数、耗时、状态码。对写操作提供 `dry_run` 参数，先验证再真实执行。

## 踩坑点

- 不要在工具描述或 prompt 里放 token、内部域名和密钥。
- 不要返回完整响应。分页接口必须裁剪、合并或限制条数。
- 写操作要带幂等键，如 `request_id` 或 `idempotency_key`，重试时复用同一键。
- 不要信任模型传入的参数。做类型、长度、枚举值校验。
- 设好循环上限，防止工具失败后 Agent 反复重试把上游打死。
- 长耗时接口不要硬等，改成提交任务返回 `task_id`，再轮询状态。

## 可复用建议

- 先接一个健康检查或 echo 工具，跑通注册、鉴权、返回全链路。
- 每个外部服务一个独立 wrapper，schema 与真实 API 解耦。
- 用环境变量统一注入 `BASE_URL`、`TIMEOUT`、`API_TOKEN`，方便切换和脱敏。
- 返回结构保持扁平稳定，字段变更要有版本号。
- 给每个工具写最小示例，帮助模型理解调用方式。

## 总结

OpenClaw 对接外部服务的核心，不是让模型学会调 API，而是把 API 封装成受控的函数工具。好的握手层具备：清晰 schema、最小权限、超时重试、错误映射、幂等和可观测。把这层做扎实，Agent 才能在噪声和异常里稳定工作。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/58f598e5ab567d63.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/0db827e8bec74448.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/9093745c9431be42.png)

