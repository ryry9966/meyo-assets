---
title: 插件排障实录：appId 数字类型导致 Gateway 异常的一次排查
feedId: 29107
source: Bug反馈
publishedAt: 2026-07-14
---

## 背景

在基于 OpenClaw 搭建的 Agent 自动化流程中，我们通过插件机制将内部系统能力暴露给 LLM。一个常见的做法是，插件注册一个工具（Tool），声明其参数 Schema，Agent 在调用时由网关（Gateway） 完成参数校验和转发。

最近在集成一个“用户查询”工具时，我们定义了一个名为 `appId` 的必选参数，业务上它是一个纯数字标识。起初，Schema 自然地写成了 `"type": "number"`。工具注册正常，单测无异常。但在一次真实请求中，Gateway 返回了 `400 Bad Request`，提示 schema validation 失败。本文梳理了整个排障过程，给出在该场景下如何避免类似类型冲突。

## 问题现象

异常日志中可以看到如下关键信息：

```
[Gateway] invalid argument: appId, reason: invalid type: string, expected number
```

这意味着请求中实际携带的 `appId` 是字符串，但 Gateway 根据工具 Schema 要求它必须是一个数字。再往前翻，Agent 生成的调用参数长这样：

```json
{
  "appId": "10086",
  "account": "someone"
}
```

而插件注册时提交给 Gateway 的 JSON Schema 片段为：

```json
{
  "name": "appId",
  "type": "number",
  "description": "应用ID"
}
```

显然，Agent 输出了带引号的数字字符串，Gateway 在校验环节拒绝了这个请求，导致整个调用链直接中断。

## 排查步骤

**1. 确认插件侧 Schema 定义**

检查插件注册代码，确认 `appId` 的确是以 `number` 类型注册的。复制出完整的 Schema，在本地用 `ajv` 等工具验证数字、字符串两种格式的输入，确认校验行为与 Gateway 日志一致 —— 字符串类型的 `"10086"` 无法通过校验，而数字 `10086` 则通过。

**2. 检查 Agent 侧 Prompt 与工具描述**

Agent 在被调用时，接收到的工具描述里只有 `type: number`，并没有说明“可以接受字符串”。然而大模型在生成 JSON 参数时，很可能因为以下几点将数字写成字符串：

- 训练语料中“ID”经常被当作文本字段处理；
- Prompt 中未强调数字必须不带引号；
- 温度采样等带来输出的不确定性。

通过打印 Agent 的原始输出，我们确认问题出在大模型生成阶段，并非网关序列化或中间件篡改。

**3. 分析 Gateway 的校验策略**

Gateway 使用的是严格的 JSON Schema 校验器，不进行隐式类型转换。这意味着只要类型不匹配，就会直接拒绝请求。这种策略保证了系统边界干净，但也要求上游（Agent）必须严格遵守 Schema。

**4. 对比类似工具的实现**

翻阅社区和其他团队分享的 OpenClaw 插件，发现多数“ID”类字段都定义为了 `"type": "string"`，即便它们是纯数字。这样做的目的是降低对 LLM 结构化输出的要求，提高调用成功率。少数坚持 `number` 的插件，通常在工具描述中额外添加了“必须为数字，不加引号”的提示，但效果仍不完美。

## 踩坑点

- **过度依赖 Schema 约束大模型输出**：JSON Schema 的 `type` 远比自然语言描述更容易被忽略。Agent 多数时候会生成合法 JSON，但不一定严格遵守 `number` / `string` 的细微区分。
- **网关校验没有灰度或兼容模式**：在严格模式下，一个字段的类型错误就会让整个调用失败，缺少降级策略。
- **业务定义与技术实现的歧义**：业务说“appId 是数字”，于是开发直接在 Schema 里写 `number`，却忽略了传输层与 LLM 生成的对齐成本。

## 可复用建议

**1. ID / 标识类字段统一使用 `string`**

在对外暴露的 API 和插件 Schema 中，凡是不会参与算术运算的“数字标识”，一律定义为 `string`。这既能适应前端输入、日志记录的习惯，也能降低 Agent 产生类型冲突的概率。

**2. Gateway 层增加兼容转换（可选）**

如果严格模式无法改变，可以在 Gateway 的预处理阶段增加一层“类型推断与修正”逻辑：对定义为 `number` 但实际收到全数字字符串的字段，尝试自动转换。这需要配合开关控制，避免掩盖真正的类型错误。

**3. 利用 Schema 的 `examples` 字段**

OpenClaw 的工具描述支持携带 `examples`，在 `appId` 的描述中增加 `"examples": ["10086"]` 可以给 Agent 明确的格式提示。实测发现，这能显著提高输出一致性。

**4. 在 Prompt 中显式提醒**

若坚持用 `number` 类型，可以在系统 Prompt 中加上一段：“所有 JSON 参数必须严格匹配 Schema 中声明的类型，数字不得加引号”。虽然不能完全避免，但能降低出错率。

**5. 上线前做多轮黑盒测试**

准备一组混合类型的测试用例（字符串数字、纯数字、带前导零的字符串等），对 Agent → 插件 → 网关整链路进行端到端验证，把类型问题挡在发布之前。

## 总结

这次排障很快，暴露的却是一个普遍的工程问题：当大模型成为调用链的一环时，我们不能再以“人类开发者遵守类型约定”的假设来设计接口。Schema 声明 `number` 看上去准确，但实际运行时，Agent 输出的不确定性会让这个准确变成脆弱的瓶颈。

一个简单的原则是：**对从 LLM 接收的入口数据，能放宽就不收紧；对标识类字段，能用 `string` 就不用 `number`。** 这条经验在 OpenClaw 插件开发、MCP 工具定义以及任何面向 Agent 的 API 设计中都同样适用。类型的一致性不应当只靠校验器来强制执行，更需要在设计方案时就对齐各参与方的默认倾向。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-14/874b27682b34ad45.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-14/f3a2f9c352cb6cb9.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-14/602733fcfedd63df.png)

