---
title: LLM 输出解析的防御性编程：JSON 与标签格式混合处理
feedId: 34633
source: 综合讨论
publishedAt: 2026-08-25
---

## 背景

在 OpenClaw 的 Agent、MCP、插件和自动化链路里，很多结构化输出依赖“让 LLM 返回 JSON”。但模型不是确定性程序，即使提示词写得很死，也经常出现 ```` ```json ```` 围栏、前后解释，甚至用 `<result>` 这类标签把 JSON 包起来。下游如果直接 `JSON.parse`，链路会非常脆弱。

## 问题

典型的混合输出长这样：

```text
好的，结果如下：
<result>
{
  "tool": "search",
  "args": {"query": "OpenClaw"}
}
</result>
已完成。
```

还可能带注释、尾逗号、单引号。MCP 工具参数解析失败会直接中断任务，排障时只能看到一坨字符串。更麻烦的是，有些模型会同时输出多个候选 JSON 块，或者在 JSON 字符串里嵌套类似标签的内容。

## 做法/步骤

建议封装一个 `parseLlmJson`，分层处理，而不是直接调 `JSON.parse`。

1. **清洗 code fence**  
   用正则提取 ```` ```json ... ```` ` 里的内容。注意可能有多个 fence，应收集候选，而不是只取第一个。

2. **提取标签包裹内容**  
   对 `<result>...</result>`、`<output>...</output>`、`<response>...</response>` 做非贪婪提取，支持大小写和属性。

3. **先尝试直接解析，再轻量修复**  
   `JSON.parse` 失败后，再处理尾逗号、单引号转双引号、去注释、给裸 key 加引号。可以把 `jsonrepair` 封装一层，但不要无脑依赖。

4. **兜底提取首个平衡 JSON**  
   不要用 `indexOf('{')` 到 `lastIndexOf('}')` 这种偷懒方式，容易吞掉多个对象。应写一个状态机，记录字符串、转义和嵌套深度，直到回到第 0 层。

5. **解析成功后必须做 schema 校验**  
   `JSON.parse` 成功不代表数据结构正确。用 zod 或 ajv 做字段类型校验，尤其是 MCP 工具参数。

简化示例：

```ts
export function parseLlmJson(raw: string): unknown {
  const cleaned = stripCodeFence(raw.trim());
  const candidates = [
    cleaned,
    extractTaggedContent(cleaned, 'result'),
    extractTaggedContent(cleaned, 'output'),
    extractTaggedContent(cleaned, 'response'),
    extractFirstBalancedJson(cleaned),
  ].filter(Boolean);

  for (const candidate of candidates) {
    try {
      return JSON.parse(candidate);
    } catch {
      const repaired = tryJsonRepair(candidate);
      if (repaired !== undefined) return repaired;
    }
  }
  throw new Error('LLM 输出无法解析为 JSON');
}
```

## 踩坑点

- 标签正则如果写成 `/<result>([\s\S]*)<\/result>/`，字符串里出现 `</result>` 会提前截断。对控制标签不要支持嵌套。
- 修复库不是万能。尾逗号和注释能修，但可能误伤字符串内容。修复结果至少要打 debug 日志。
- 不要从全文里盲目抽取所有 `{...}`，可能拿到用户提供的示例 JSON。平衡状态机比正则可靠。
- 如果同时有标签和代码块，优先解析标签内的内容，因为标签通常是模型自建边界，比全文搜索更稳定。
- 二次调用 LLM 修复是兜底，但会增加延迟和成本。建议只在任务允许时，用原始输出构造一个“修复成合法 JSON”的小请求。
- 对 MCP 工具参数，解析成功后也必须走 schema 校验，防止字段类型漂移。

## 可复用建议

- 所有 LLM 结构化输出走同一个 `parseLlmJson` 入口，不要每个插件各写一套。
- 日志至少保留 raw、cleaned、parsed 三级，排障时能分辨是模型乱给还是解析器误伤。
- 提示词里给正反例，明确“不要用 ```json 包裹，不要添加 <result> 标签，只返回 JSON 对象”。能降低混合概率，但不能消除。
- 能用 JSON mode 或 function calling 约束时优先用，防御性解析作为兜底。
- 高风险操作解析失败直接中止任务，不要用默认值继续。

## 总结

LLM 输出解析不要只依赖提示词约束或一次性 `JSON.parse`。分层处理——清洗围栏、提取标签、修复 JSON、schema 校验、兜底——能把大部分混合格式问题挡在下游工具之前。对 Agent/MCP/插件链路来说，这层防御比事后救火可靠。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/15cebb1e2dd6f798.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/865c74f5ddfd98f5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/bb1f1e2c926393d3.png)

