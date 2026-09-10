---
title: Agent 与 API 的握手：OpenClaw 怎么对接外部服务
feedId: 36938
source: 综合讨论
publishedAt: 2026-09-10
---

## 背景

用 OpenClaw 做过一点实际事情的都知道：agent 本体只负责推理和决策，真正干活的是它挂载的 tools / skills。无论是查订单、调搜索、触发 CI，还是读写内部系统，本质都是同一件事——把一个 HTTP API 翻译成 agent 能稳定使用的工具。OpenClaw 提供两条接入路径：MCP server 和原生插件（tool 定义直接写在配置里），本文不纠结选型，聚焦握手本身。

## 问题

实际翻车的场景很少是"连不上"，更多是"连上了但 agent 用不好"：

- 工具描述含糊，agent 乱传参数或在该调用时不调用
- API key 硬编码进配置，或被完整打进日志
- 没有超时和重试策略，一次网络抖动让 agent 幻觉式总结"服务已下线"
- 原始返回体太大，几次调用就撑爆上下文

## 做法

以把一个内部订单查询 API 接成 tool 为例，步骤如下：

**1. 定义 schema。** name 用动词短语（`query_order`），参数给显式类型、enum、required。description 是写给 agent 看的文档，重点写"什么时候该用、什么时候不该用"：

```yaml
name: query_order
description: 当用户询问订单状态或物流时调用；输入订单号；
  查不到返回 not_found，此时应告知用户而非重试。
parameters:
  order_id: { type: string, required: true }
  status_filter: { type: string, enum: [all, shipped, pending] }
```

**2. 写 connector 层。** 插件里只做三件事：参数校验、HTTP 调用、返回裁剪。不要把原始 JSON 整个丢回上下文——挑关键字段、截断长文本、必要时补分页参数。

**3. 凭证管理。** key 走环境变量或 secret 文件，配置里只留引用名，日志统一脱敏。

**4. 错误语义映射。** 这是最容易被忽略的一步。"订单不存在"和"服务暂时不可用，可稍后重试"在 agent 眼里必须是两种信号，后者要带 retryable 标记，否则 agent 会把临时故障当永久事实。

**5. 超时与重试。** 连接超时 3–5s，总超时按业务定；只对幂等请求重试，指数退避，上限 2 次。重试决策放在 connector 里，不要让 agent 自己决定——它没有判断重试成本的上下文。

**6. 单测再接入。** 用调试模式固定输入跑这个 tool，先看原始返回是否符合预期，再挂进 agent 流程。

## 踩坑点

- description 写成给人看的 API 文档是常见错误。"查询订单"不如"用户问到订单状态或物流时调用，输入订单号，查不到返回 not_found"。
- 日期、时区类参数出错率最高，schema 里直接给格式示例。
- 返回裁剪别过头，字段名要保留语义，否则 agent 等于瞎猜。
- 对接免费额度 API，上线后被 agent 循环调用打爆限流，记得在 connector 侧加自己的限流。
- 路径选择：需要动态发现、多客户端复用选 MCP server；单一内部服务追求低延迟，直接写插件更省事。

## 可复用建议

- 沉淀 tool 四件套模板：schema + connector + 错误映射 + 测试用例，新 API 照抄结构，半小时接入。
- 所有外部调用收敛到一个共享 HTTP client，统一超时、重试、脱敏日志，别每个插件各写一套。
- 给每个 tool 记录调用指标（成功率、p95 延迟）。agent 行为异常时，先看这里，再看提示词。

## 总结

对接外部服务的难点不在 HTTP，而在"翻译"：schema 是契约，connector 是边界，错误语义是体验。把这一层做扎实，agent 才不会在每次握手上掉链子。欢迎在评论区交流你们踩过的坑。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/4a8e1711f19563a1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/a968a8f7852d4cfa.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/f40b4a4df2653429.png)

