---
title: 防御性解析 LLM 输出：JSON、代码块与自定义标签混合处理
feedId: 34738
source: 综合讨论
publishedAt: 2026-08-26
---

## 背景

在 OpenClaw 的 Agent、MCP 工具和插件链路里，经常需要让模型返回结构化数据。即使提示词明确要求“只输出 JSON”，线上仍然会看到多种混合形态：

- ```` ```json { ... } ``` ````
- `<json>{ ... }</json>` 或 `<result>{ ... }</result>`
- “结果如下：{ ... }”，前后附带自然语言说明
- 一段输出里出现多个 JSON 候选，其中只有一个是真正需要的结果

如果系统直接 `JSON.parse(raw)` 或 `json.loads(raw)`，失败率会明显高于预期。防御性解析的目标不是“保证一次成功”，而是让失败可恢复、可诊断、不直接击穿整个流程。

## 问题

真实 LLM 输出不是固定格式，而是 JSON、Markdown 代码围栏、自定义标签和自然语言的混合体。只靠一个强正则，比如 ```` ```json([\s\S]*?)``` ````，很容易在以下情况失效：

- 代码块没有闭合
- 自定义标签不是 ```` ```json ````，而是 `<json>`、`<output>`、`[JSON]`
- JSON 字符串内部包含 `{`、`}` 或反引号
- 返回了多个 JSON 对象
- 存在尾逗号、注释、BOM、零宽字符

解析器一旦抛错，上层 Agent 可能直接判定任务失败，触发昂贵重试，甚至丢弃上下文。

## 做法/步骤

比较稳的实践是分层降级，同时保留原始输出。

### 1. 保留 raw

任何清洗动作之前，先记录原始字符串、长度、模型名和时间。这会让后续排障有据可查，而不是面对一个已经处理过的空串。

### 2. 预处理

去除 BOM、零宽字符，统一换行。注意不要过早删除标点或反引号。

### 3. 提取候选区

先剥掉已知包装：

- Markdown 代码围栏：优先完整围栏，再处理只有开头 ```` ``` ```` 但未闭合的情况
- 自定义标签：`<json>...</json>`、`<output>...</output>`、`<result>...</result>`、`[JSON]...[/JSON]`
- 对标签提取使用非贪婪匹配前，先判断闭合标签是否存在

然后做括号扫描：从第一个 `{` 或 `[` 开始，维护深度计数，遇到字符串进入/退出，直到深度归零。正则无法可靠解决字符串内括号问题。

### 4. 候选排序

同一段 raw 可能提取到多个候选。优先选择“长度适中 + schema 校验通过 + 字段完整”的候选，而不是盲目选第一个。

### 5. 解析降级

按顺序尝试：

1. 严格 JSON 解析
2. 去尾逗号、去注释后重试
3. 可选 JSON5 或宽松解析
4. 仍失败时进入一次性 LLM repair，把原始输出和解析错误交给模型，要求只返回修正后的 JSON

LLM repair 应设置次数上限，建议只做 1 次，避免无限重试或成本失控。

### 6. Schema 校验

类型、必填字段、枚举值都要校验。解析成功不等于业务数据正确，只有校验通过才允许进入后续工具调用、数据库写入或插件执行。

一个小型 TypeScript 提取示例如下：

```ts
function extractJsonCandidates(raw: string): string[] {
  const clean = raw
    .replace(/^\uFEFF/, '')
    .replace(/```(?:\w+)?\s*([\s\S]*?)```/g, '$1')
    .replace(/<json>([\s\S]*?)<\/json>/gi, '$1')
    .replace(/<output>([\s\S]*?)<\/output>/gi, '$1')

  const out: string[] = []
  for (const start of ['{', '[']) {
    let i = clean.indexOf(start)
    while (i !== -1) {
      const slice = scanBalanced(clean, i)
      if (slice) out.push(slice)
      i = clean.indexOf(start, i + 1)
    }
  }
  return out
}
```

其中 `scanBalanced` 是核心实现，负责在避开字符串内容的前提下完成括号匹配。

## 踩坑点

- **`indexOf('{')` 会命中字符串内容**：如 `"text": "价格 {discount}"`，必须用括号扫描，而不是简单截取。
- **非贪婪正则遇到嵌套或缺失闭合会漏**：标签包裹不完整时直接返回空，可能丢失后续候选。应先判闭合，再降级到括号扫描。
- **不要删除所有反引号**：JSON 字符串里本身可能包含反引号。应只剥代码围栏。
- **多 JSON 块**：模型可能输出“示例 + 真实结果”，应选最大或 schema 最匹配的候选，而不是第一个。
- **JSON5 是双刃剑**：允许单引号、尾逗号和注释能提升成功率，但也可能掩盖输出问题。建议只在可控场景开启，并记录实际使用的 parser。
- **`null`、`true`、数字也是合法 JSON**：如果业务只接受对象或数组，schema 校验必须限制根类型。
- **LLM repair 要留痕**：记录原始输出、错误信息和修复结果，否则排障时不知道是提取失败还是模型本身输出错误。

## 可复用建议

建议在你的插件或 MCP 工具里把解析流程收敛为一个函数：

```ts
parseLooseJson(raw, schema, opts)
```

返回结构可以统一为：

```ts
{
  ok: boolean,
  data?: any,
  raw: string,
  extracted?: string,
  parser?: string,
  error?: string
}
```

这样上层 Agent 能清楚区分“模型输出为空”“JSON 截断”“schema 不匹配”等不同情况。

同时，提示词层可以继续要求“只输出 JSON，不要代码块，不要解释”，但解析层永远不要信任提示词约束。真实工程里，模型输出不可控，必须按不可信输入处理。

## 总结

JSON、标签和代码块混合，是 LLM 输出解析中的常态，不是偶发 bug。防御性编程的核心不是写一个大而全的正则，而是把“提取、解析、校验、修复、留痕”拆开，每一层都能独立失败和降级。对 OpenClaw、Agent、MCP 和自动化插件来说，这样的做法能显著减少流程中断，也让线上问题更容易定位。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/bb50adc830cadbec.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/b09f1da8408c8d2b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/cd3ea14e60b24880.png)

