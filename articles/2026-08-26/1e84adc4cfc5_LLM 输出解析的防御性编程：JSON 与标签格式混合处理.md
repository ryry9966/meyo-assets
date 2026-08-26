---
title: LLM 输出解析的防御性编程：JSON 与标签格式混合处理
feedId: 34834
source: 综合讨论
publishedAt: 2026-08-26
---

## 背景

在 OpenClaw、Agent、MCP 以及插件/自动化链路里，我们经常让 LLM 返回 JSON，然后直接交给下游工具调用或配置解析。看起来很简单：模型说好输出 JSON，我们用 `JSON.parse` 接住。

实际运行一段时间后，输出形态会非常不稳定。常见的有：

```text
```json
{"tool": "search", "args": {"q": "status"}}
```
```

也有包在标签里的：

```text
<result>
{"tool": "search", "args": {"q": "status"}}
</result>
```

更麻烦的是混合形态：

```text
好的，结果如下：
<json>
```json
{"tool": "search", "args": {"q": "status"}}
```
</json>
```

如果解析层只做 `JSON.parse(raw)`，这三种情况都会直接抛异常，进而导致 Agent 任务中断、MCP 工具返回 500，或者插件把原始报错透传给用户。这个问题不能只靠 prompt 约束，必须在解析层做防御性处理。

## 问题

真正的难点不是某一种格式，而是“混合”。例如：

- 标签内嵌套 Markdown code fence；
- 模型前后附加解释性文字；
- 输出里同时出现多个 JSON 对象；
- 标签大小写不一致，比如 `<JSON>`、`<Result>`；
- 不可见字符、BOM、零宽空格混在 JSON 前面。

如果只写一个简单正则，很容易在某个真实 case 上截错位置。

## 做法/步骤

我建议把解析拆成三层：候选提取、解析修复、schema 校验。

### 1. 先做基础清洗

不要一上来就看 `{`，先把不可见字符去掉：

```ts
const cleaned = raw
  .replace(/^\uFEFF/, '')
  .replace(/[\u200B-\u200D\uFEFF]/g, '')
  .trim();
```

### 2. 按优先级提取候选 JSON

优先级建议：

1. Markdown code fence；
2. 显式标签，如 `<json>`、`<result>`、`<output>`；
3. 裸对象：从第一个 `{` 到最后一个 `}`。

```ts
function extractCandidates(text: string): string[] {
  const candidates: string[] = [];

  for (const m of text.matchAll(/```(?:json|jsonc)?\s*([\s\S]*?)```/gi)) {
    if (m[1]?.trim()) candidates.push(m[1].trim());
  }

  for (const m of text.matchAll(/<(?:json|result|output)\b[^>]*>([\s\S]*?)<\/(?:json|result|output)>/gi)) {
    const inner = m[1]?.trim();
    if (!inner) continue;
    const fenced = inner.match(/```(?:json|jsonc)?\s*([\s\S]*?)```/i);
    candidates.push(fenced ? fenced[1].trim() : inner);
  }

  const firstBrace = text.indexOf('{');
  const lastBrace = text.lastIndexOf('}');
  if (firstBrace !== -1 && lastBrace > firstBrace) {
    candidates.push(text.slice(firstBrace, lastBrace + 1));
  }

  return candidates;
}
```

这里注意正则用了非贪婪 `*?`，避免多个对象时直接吞到最后一个。

### 3. 逐候选尝试解析，再决定是否 repair

```ts
function parseLlmJson(raw: string): unknown | null {
  const candidates = extractCandidates(raw);

  for (const cand of candidates) {
    try {
      return JSON.parse(cand);
    } catch {
      // 可选用 jsonrepair 修复单引号、尾逗号、注释等常见问题
      // const repaired = jsonrepair(cand);
      // try { return JSON.parse(repaired); } catch {}
    }
  }

  return null;
}
```

不建议无脑 repair 所有候选。`jsonrepair` 是启发式修复，可能把错误候选“修”成一个语义错误的 JSON。更稳妥的是先尝试严格 parse，再 repair。

### 4. 用 schema 验证，而不是只看 parse 成功

解析成功不代表字段可用。比如模型返回了 `{"name": "..."}`，但工具需要 `tool` 和 `args`。所以解析后要再做校验：

```ts
import { z } from 'zod';

const ToolCallSchema = z.object({
  tool: z.string(),
  args: z.record(z.unknown()).default({}),
});

const parsed = parseLlmJson(raw);
const result = ToolCallSchema.safeParse(parsed);
```

如果 `safeParse` 失败，说明候选可能选错了。这时可以回到候选列表，尝试下一个候选，而不是直接丢弃。

### 5. 返回结构化错误与回退

解析失败时，不要抛原始字符串，也不要 `return null` 后让上层自己猜。建议返回一个可判断的结构：

```ts
type ParseResult =
  | { ok: true; data: unknown }
  | { ok: false; reason: string; raw: string };
```

上层拿到 `ok: false`，可以选择把 `reason` 和原始输出回传给模型，要求重新输出；也可以走本地降级逻辑。这样整条链路不会被一个异常打断。

## 踩坑点

- **标签大小写**：`<JSON>`、`<Result>` 都会出现，正则要加 `i` flag。
- **标签内再套 fence**：不能只匹配标签内文本，还要继续提取 code fence。
- **多个 JSON 候选**：模型举例说明时可能输出两个对象，按 schema 验证筛选比取第一个更可靠。
- **裸对象误抓**：如果文本里有解释性文字，例如“像 `{"a":1}` 这样”，裸对象候选可能不是目标。优先 code fence 和标签，可以降低误判。
- **不可见字符**：BOM 和零宽空格会导致 `JSON.parse` 失败，但肉眼完全看不出来。
- **不要过度依赖 response_format**：即使开启 JSON 模式，部分网关或模型仍可能包裹 Markdown 或标签。
- **repair 有副作用**：修复工具会改内容，可能掩盖真实格式错误，建议只作为最后的兜底，并配合 schema 校验。

## 可复用建议

1. 把 `parseLlmJson` 放到公共 utils，所有 MCP 工具、Agent action、插件配置解析统一调用。
2. 解析层返回 `ok/data/reason/raw` 四元组，不要让异常流进入业务代码。
3. 先严格 parse，再 repair，再做 schema 校验，顺序不要反。
4. 在 prompt 里明确输出格式，但不要把 prompt 当成唯一保障。
5. 记录原始输出片段和命中的候选 index，方便排障。至少日志里保留前 200 个字符，不要只记“parse failed”。

## 总结

LLM 输出解析不是简单的 `JSON.parse`，而是一层独立的防御性工程。将 code fence、标签、裸对象混合处理，先提取候选，再解析、修复、校验，最后结构化返回错误。这样 Agent/MCP/插件链路才能在模型输出不稳定时仍然可控、可重试、可降级。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/46be842804af0562.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/2a6f7aaa66988cd1.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/15e7b6f7923a34ee.png)

