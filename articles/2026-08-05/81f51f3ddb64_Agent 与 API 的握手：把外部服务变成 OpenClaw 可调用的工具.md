---
title: Agent 与 API 的握手：把外部服务变成 OpenClaw 可调用的工具
feedId: 31664
source: 综合讨论
publishedAt: 2026-08-05
---

## 背景

Agent 的能力边界取决于它能触达的工具数量。OpenClaw 的核心定位是让 agent 在命令行、浏览器、桌面环境之间自由行动，但真正让它产生业务价值的，往往是那些部署在远程的外部 API——订单系统、资产查询、内部运维接口。

MCP 解决的是协议标准化问题，但工程落地时你会发现：接入 MCP server 只是第一步，如何让 agent 在正确时机、以正确参数调用这些服务，才是握手成功的关键。

## 问题

OpenClaw 本身支持多种工具加载方式，遇到外部 API 时，开发者通常面临三个选择：

1. 直接写死 HTTP 请求，让 agent 按固定 prompt 调用 —— 灵活但不可控，参数出错无法感知。
2. 引入 MCP server 包装 —— 标准但偏重，对简单只读接口来说额外维护一个 service 成本偏高。
3. 使用 OpenClaw 的 service connector 机制，把外部 API 注册为原生工具 —— 推荐，这能让 agent 像调用内置 skill 一样使用你的服务。

对多数团队而言，第三种方案最务实，但落地细节里有不少坑。

## 做法

以下是我们在 OpenClaw 中对接一个内部资产查询 API 的完整步骤，供参考。

**第一步：写 service connector**

在 OpenClaw 的 services 目录下新增一个 Python connector，核心是声明服务的基础信息和调用方式：

```python
from openclaw.services import ServiceConnector, tool

class AssetService(ServiceConnector):
    name = "asset_service"
    base_url = "https://api.internal.example.com"

    @tool(description="按关键词搜索资产信息")
    def search(self, keyword: str, limit: int = 10) -> dict:
        return self._request("/asset/search", params={"q": keyword, "n": limit})
```

**第二步：定义清晰的 schema 与返回结构**

这一步容易被忽视。`@tool` 装饰器生成的 schema 是 agent 理解的唯一依据，描述务必具体到参数含义，返回值尽量扁平化。嵌套过深的结构会让 agent 在后续推理中丢失信息。

**第三步：注册进能力清单**

在 OpenClaw 的配置文件中启用该 connector，并绑定到对应的 agent profile。如果不做这一步，agent 可能永远不会主动调用它——尤其是默认关闭的 service。

**第四步：调试确认**

用 `--debug` 模式跑一轮交互，检查两点：

- agent 是否能从对话上下文中正确提取参数并触发调用；
- 返回结果是否能被 agent 用自然语言转述给用户。

## 踩坑点

**1. timeout 设置不当导致 agent "沉默"**

外部 API 响应超过 5 秒，agent 大概率会认为工具不可用，转而回答"暂无法获取"。保证 connector 内所有请求都有明确且偏短的 timeout，并在异常分支里返回可读错误信息，不要让异常穿透到 agent 上下文。

**2. 过度暴露工具**

第一次接入时把所有 API 方法都挂成 `@tool`，结果 agent 面对 30 多个工具，决策质量明显下降。工具数量控制在 5 个以内时，选择准确率最高。合并参数相近的方法，把低频方法挪到 secondary 目录。

**3. schema 命名冲突**

MCP server 名相同但路由不同的两个服务，在 OpenClaw 里会被视为同一 namespace 的工具，容易互相覆盖。务必给每个 connector 起全局唯一的名字，并在工具描述里写明出处。

## 可复用建议

- **窄接口优先**：每个工具只做一件事，参数少，返回短。这比一个万能接口更利于 agent 推理。
- **幂等设计**：外部 API 尽量支持幂等，重试不会产生重复副作用，这样 agent 才能在超时后安全重试。
- **日志追踪**：在 connector 里给每次调用加 trace_id，便于事后排查 agent 的调用路径和失败原因。
- **回归测试**：每次改动外部 API 的字段名或 schema，都要重新跑一轮已有的 agent 测试用例，避免静默破坏。

## 总结

Agent 与 API 的握手，本质是"用工程约束换取模型自由度"。明确 schema、控制工具数量、做好异常兜底，OpenClaw 的 agent 才会真正成为你可以依赖的自动化执行者，而不是一个偶尔调对接口的玩具。

---

