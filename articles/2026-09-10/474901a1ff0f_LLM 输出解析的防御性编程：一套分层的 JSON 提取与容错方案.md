---
title: LLM 输出解析的防御性编程：一套分层的 JSON 提取与容错方案
feedId: 36843
source: 综合讨论
publishedAt: 2026-09-10
---

## 背景

在 OpenClaw 插件、Agent 工作流和 MCP 工具链里，让模型输出结构化 JSON 是高频操作：节点间传参、工具调用结果落库、插件配置抽取。提示词里写「只输出 JSON，不要任何多余内容」，多数时候模型会配合——但「多数时候」放到生产环境，约等于每天都会失败几次。

## 问题

真实输出是一个分布，不是一个确定值。常见的变体包括：

1. 标准 ```` ```json ```` 围栏包裹；
2. 裸 JSON，什么都没有；
3. 前置寒暄：「好的，以下是解析结果：」+ JSON；
4. JSON 之后跟一段解释文字；
5. 多个代码块（一个示例 + 一个真结果）；
6. XML 风格标签包裹：`<result>{...}</result>`，有时还混着思考块；
7. 近似 JSON：尾逗号、单引号、中文引号、`//` 注释；
8. JSON 字符串值里本身含 ```` ``` ```` 或 `}`（嵌套代码场景）。

用一条正则加 `JSON.parse` 硬上，结果就是「本地测试全过，上线三天两崩」。

## 做法：分层解析管线

核心思路：不要指望某一招通吃，把解析做成逐层降级的管线，每层只处理自己擅长的情况，并把降级事件记录下来。

```ts
function parseModelJson(raw: string): unknown {
  // L0：直接解析
  try { return JSON.parse(raw); } catch {}

  // L1：剥围栏；没有围栏再试 <result>/<json> 等标签包裹
  const fenced = raw.match(/```(?:json|jsonc|json5)?\s*([\s\S]*?)```/);
  const tagged = raw.match(/<(?:result|json|output)[^>]*>([\s\S]*?)<\/\w+>/);
  for (const c of [fenced?.[1], tagged?.[1]]) {
    if (c) { try { return JSON.parse(c); } catch {} }
  }

  // L2：字符串感知的括号配对提取（应对前后缀噪声、多块输出）
  const extracted = extractBalanced(raw);
  if (extracted) { try { return JSON.parse(extracted); } catch {} }

  // L3：近似 JSON 修复，最后手段
  const repaired = repair(extracted ?? fenced?.[1] ?? raw);
  if (repaired) { try { return JSON.parse(repaired); } catch {} }

  throw new ParseError(raw);
}
```

L2 是关键。常见错误是写贪婪的 `/{.*}/`，或者裸数花括号。正确做法是逐字符扫描，进入字符串状态时（处理 `\"` 转义）不计数括号：

```ts
function extractBalanced(s: string): string | null {
  const start = s.search(/[{[]/);
  if (start < 0) return null;
  let depth = 0, inStr = false, esc = false;
  for (let i = start; i < s.length; i++) {
    const c = s[i];
    if (inStr) {
      if (esc) esc = false;
      else if (c === '\\') esc = true;
      else if (c === '"') inStr = false;
    } else {
      if (c === '"') inStr = true;
      else if (c === '{' || c === '[') depth++;
      else if (c === '}' || c === ']') {
        if (--depth === 0) return s.slice(start, i + 1);
      }
    }
  }
  return null;
}
```

（严格场景下 `{}` 与 `[]` 应用栈做配对，思路一致，篇幅所限从简。）

解析成功不等于数据正确。外面再套一层 schema 校验（zod/ajv），失败时把「原始输出 + 具体错误信息」拼回提示词重试，设置上限（比如 2 次），避免无限循环烧 token。

## 踩坑点

- **贪婪正则** `/{.*}/` 会吞掉多个对象之间的所有内容，多块输出时必然出错。
- **数括号不看字符串状态**：值里有 `"}"`（比如代码片段）就会提前截断。
- **修复要保守**：单引号转双引号可能破坏值里的撇号。修复只作用于「已提取的片段」，不要对原始全文乱来，并始终保留原始输出便于回溯。
- **修复不要静默**：每次降级都打日志或上报。否则管线「看起来很稳」，实际在掩盖提示词劣化。
- **重试无上限 = 账单无上限**：解析失败重试要和工具调用重试分开计数。

## 可复用建议

- 把这套管线做成共享 util 或独立插件，不要每个 skill 各抄一份——降级策略不一致会让排查变成灾难。
- 统计每一层的命中率：如果 L3 频繁触发，优先改提示词或收紧 schema，而不是继续加固修复逻辑。
- 运行时支持 structured output / function calling 就优先用，但保留解析兜底：经手 MCP 代理或模型中转时，格式保障并不总是可靠。
- 把线上真实的坏输出攒成 fixture 语料，解析器每次改动跑一遍回归，比任何模板化单测都有用。
- 结构化任务压低 temperature，格式漂移会明显减少。

## 总结

防御性解析不是不信任模型，而是承认「输出格式是一个带噪声的分布」。把解析当成有可观测性的管线：严格路径优先、逐层降级、修复保守、失败可反馈。做到这几点，结构化输出在自动化链路里才敢说「基本不用人盯」。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/3c5cc868983ba13c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/0757724f696f0a8a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/85262f4fcb304a4c.png)

