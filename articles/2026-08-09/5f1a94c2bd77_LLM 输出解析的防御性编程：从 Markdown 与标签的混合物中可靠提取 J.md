---
title: LLM 输出解析的防御性编程：从 Markdown 与标签的混合物中可靠提取 JSON
feedId: 32247
source: 综合讨论
publishedAt: 2026-08-09
---

# 背景

在 OpenClaw 搭建的自动化工作流、MCP 工具链或自定义 Agent 里，我们经常要求大模型返回结构化数据——大多数情况下就是 JSON。理想中只需设置 `response_format: json_object` 就能得到规整的结果，但现实很骨感：

- 模型会在 JSON 外面包裹 ```` ```json ```` 这样的 Markdown 代码块标记；
- 前面加一句解释性的自然语言，再跟上 JSON；
- 甚至输出多个 JSON 块，或者夹带一些“注释”风格的描述（`// 注意这是类型A`）；
- 响应被截断、里面多了一个尾逗号、使用了单引号等等。

如果直接用 `JSON.parse`，几乎注定会在某个生产场景里崩溃。防御性编程不是过度设计，而是让整个自动化链路不会因为一个“差不多能用”的解析逻辑而挂掉。

# 问题

假设我们通过 OpenClaw 的自定义 Agent 调用一个远端模型，希望它返回这样的结构：

```json
{
  "action": "search",
  "query": "latest AI news"
}
```

但实际拿到的可能类似：

```
好的，我将为您执行搜索。
```json
{
  "action": "search",
  "query": "latest AI news", // 这里是备注
}
```
```

这段文本里混入了：
1. 前缀说明文字；
2. Markdown 代码块标记；
3. JSON 中带有非法尾逗号和类注释。

直接 `JSON.parse` 会抛出 `SyntaxError`，然后 Agent 流程中断。我们需要一种健壮、可组合的提取与修复策略。

# 做法 / 步骤

我提炼出一个适合在 Node.js 环境（OpenClaw 插件常用）使用的 `parseLLMOutput` 流程。核心分三步：**提取候选文本 → 修复非标 JSON → 解析**。

### 1. 去除 Markdown 代码包装

先用正则把最常见的 ```` ```json ```` 或 ```` ``` ```` 包裹的代码块内容取出。注意可能有多个代码块，或者代码块嵌套在普通文本里。我采用从第一个 ```` ``` ```` 开始到最后一个 ```` ``` ```` 结束的思路，如果存在代码块，就取其中内容；否则继续使用原始文本。

```ts
function stripCodeFence(text: string): string {
  const fenceMatch = text.match(/```(?:json)?\s*([\s\S]*?)\s*```/);
  return fenceMatch ? fenceMatch[1].trim() : text;
}
```

### 2. 提取平衡括号区域

经过上一步之后，可能还有前缀/后缀的自然语言。JSON 对象以 `{` 开头、`}` 结尾；数组以 `[` 开头、`]` 结尾。我们通过扫描找到第一个 `{` 或 `[`，然后维护一个括号栈，忽略字符串内的括号，得到最外层完整的结构。这个步骤能扔掉多余的前后文。

```ts
function extractBalancedJSONBlock(raw: string): string | null {
  const startIndex = Math.min(
    raw.indexOf('{') === -1 ? Infinity : raw.indexOf('{'),
    raw.indexOf('[') === -1 ? Infinity : raw.indexOf('[')
  );
  if (startIndex === Infinity) return null;

  const stack: string[] = [];
  let inString = false;
  let escaped = false;
  for (let i = startIndex; i < raw.length; i++) {
    const char = raw[i];
    if (escaped) { escaped = false; continue; }
    if (char === '\\') { escaped = true; continue; }
    if (char === '"') { inString = !inString; continue; }

    if (!inString) {
      if (char === '{' || char === '[') {
        stack.push(char);
      } else if (char === '}' || char === ']') {
        const open = stack.pop();
        if (
          (char === '}' && open !== '{') ||
          (char === ']' && open !== '[')
        ) {
          // 括号不匹配，放弃
          return null;
        }
        if (stack.length === 0) {
          return raw.slice(startIndex, i + 1);
        }
      }
    }
  }
  return null; // 没有闭合
}
```

### 3. 修复并解析

取得候选字符串后，我会先尝试直接解析。失败则用专门的 JSON 修复库（如 `json-fixer` 或 `json-repair`）处理尾逗号、单引号、缺失引号等问题，再解析一次。如果仍然失败，记录原始内容和错误，触发告警，但不让流程崩溃——返回一个安全的 fallback 或者 `null`。

```ts
import { jsonrepair } from 'jsonrepair';

function safeParse(candidate: string): unknown | null {
  try {
    return JSON.parse(candidate);
  } catch {
    try {
      const repaired = jsonrepair(candidate);
      return JSON.parse(repaired);
    } catch (e) {
      // 记录日志供排查
      console.error('JSON repair failed', e, candidate.slice(0, 200));
      return null;
    }
  }
}
```

将这些串起来：

```ts
function parseLLMOutput(text: string): unknown | null {
  const withoutFence = stripCodeFence(text);
  const candidate = extractBalancedJSONBlock(withoutFence) || withoutFence;
  return safeParse(candidate);
}
```

# 踩坑点

- **警惕字符串内括号**：`extractBalancedJSONBlock` 必须正确处理转义和引号，否则 `{"text": "}"}` 会被截断。上述实现用 `inString` 和 `escaped` 做了防护，实测稳定。
- **多个 JSON 对象同时出现**：模型可能输出 `{"a":1} {"b":2}`。此时 `extractBalancedJSONBlock` 只会返回第一个，其余的会被忽略。若业务需要处理多个对象，可循环提取，但要注意防止死循环——用 `while` 每次提取后移动 `startIndex`。
- **修复库不是银弹**：对语义错误的 JSON（如将数组写成对象），修复是徒劳的。应把 `jsonrepair` 视为纯语法层面的纠正，不能过分依赖。
- **换行与空白干扰**：部分模型输出时在 JSON 前后加全角空格或零宽字符。预先做一次 `trim` 并用 `/^[\s\uFEFF\xA0]+/` 清理开头是必要的。
- **截断输出**：若完整 JSON 被截断，`extractBalancedJSONBlock` 会返回 `null`，此时可以退而求其次，使用一个 JSON 前缀解析器，但多数场景下宁可失败重试，也不要用不完整数据。

# 可复用建议

将上述方案封装成一个独立的工具函数，放入 OpenClaw 的 `utils` 或 MCP 工具的预处理步骤中。对于任何需要解析 LLM 返回 JSON 的场景（工具调用参数、结构化回答、数据提取），调用 `parseLLMOutput` 代替裸 `JSON.parse`。配合日志，能大幅降低线上因“Unexpected token”造成的 Action 中断。

如果在 OpenClaw 中使用 `structured_output` 功能，可以要求模型返回 Function Call 风格的 JSON，但依然不能 100% 避免格式混乱，因此建议保留这层防御。

# 总结

在 Agent 和自动化链路中，LLM 输出的稳定性不能只靠 Prompt 或模型能力。用低成本的防御性解析逻辑，把 Markdown 标签、自然语言前缀和非标 JSON 这些常见的“脏”数据消解掉，能显著提升鲁棒性。上述方法仅用了少量代码，但涵盖了代码块去除、括号提取、JSON 修复三个关键环节，可以作为一个标准范式复用到任何需要解析模型输出的地方。

生产环境里，多几行防御代码，就少一个凌晨3点的告警。

---

