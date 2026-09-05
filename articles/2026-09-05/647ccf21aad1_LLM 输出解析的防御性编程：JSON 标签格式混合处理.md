---
title: LLM 输出解析的防御性编程：JSON 标签格式混合处理
feedId: 36224
source: 综合讨论
publishedAt: 2026-09-05
---

## 背景

在 OpenClaw 的插件和自动化链路里，LLM 输出经常要被当结构化数据用：抽字段、喂给 MCP 工具、写配置。我们在 prompt 里写了"只输出 JSON"，模型大多数时候也确实照做了——但"大多数时候"放进自动化流程，等于必然出错。解析是概率性输出和确定性代码之间的边界，这个边界必须按不可信输入设防。

## 问题：解析失败的典型形态

一条链路跑多个模型（或同一模型不同温度），返回格式五花八门：

1. 裸 JSON；
2. ` ```json ` 围栏包裹；
3. 围栏无语言标注，或前后带空行、BOM；
4. 前后混解释性文字："好的，以下是结果：{...}"；
5. JSON 含尾逗号、中文引号、未转义换行；
6. 偶发自创标签 `<answer>{...}</answer>`。

直接 `JSON.parse` 从第 2 条起全挂，且失败点随机，日志里只有一句 `Unexpected token`，很难定位。

## 做法：五步分层

**1. 提取优先于解析。** 先剥围栏，再扫描首个括号配平片段。括号计数必须感知字符串，否则 JSON 字符串里的 `{` 会破坏深度：

```js
function extractJSON(raw) {
  const text = raw.trim().replace(/^\uFEFF/, '');
  const candidates = [];
  const fence = text.match(/^```(?:json)?\s*([\s\S]*?)\s*```$/);
  if (fence) candidates.push(fence[1]);
  const start = text.search(/[{[]/);
  if (start !== -1) {
    const open = text[start], close = open === '{' ? '}' : ']';
    let depth = 0, inStr = false, esc = false;
    for (let i = start; i < text.length; i++) {
      const c = text[i];
      if (inStr) {
        if (esc) esc = false;
        else if (c === '\\') esc = true;
        else if (c === '"') inStr = false;
        continue;
      }
      if (c === '"') inStr = true;
      else if (c === open) depth++;
      else if (c === close && --depth === 0) {
        candidates.push(text.slice(start, i + 1)); break;
      }
    }
  }
  candidates.push(text); // 兜底：裸输出
  return candidates;
}
```

**2. 候选依次严格解析。** 顺序：围栏内容 → 配平片段 → 全文。严格解析永远先行，"修复"只对失败的候选做。

**3. 受控修复。** 只做确定性高的替换：尾逗号、中文引号、零宽字符。别上大而全的"智能修复"——修复逻辑本身也可能把合法数据改坏。

```js
function parseLoose(raw) {
  for (const c of extractJSON(raw)) {
    try { return JSON.parse(c); } catch {}
    try {
      return JSON.parse(c
        .replace(/,\s*([}\]])/g, '$1')   // 尾逗号
        .replace(/[“”]/g, '"')           // 中文引号
        .replace(/[\u200b\u200c]/g, ''));// 零宽字符
    } catch {}
  }
  throw new Error('unparsable');
}
```

**4. Schema 校验。** parse 成功 ≠ 结构正确。用 zod/ajv 校验字段，错误信息具体到字段名。

**5. 带反馈的重试。** 解析或校验失败时，把错误原因和上次输出截断后回传："你的输出因 X 无法解析，请只输出一个 JSON 对象，不要围栏。"两轮内多数能救回来；仍失败就落盘原始输出并告警，不要静默吞掉。

## 踩坑点

- `/\{.*\}/` 贪婪匹配是最常见的坑，遇到"文本里嵌代码示例再嵌 JSON"会把两个对象缝成一个；
- 括号计数不处理转义（`\"`）会提前截断；
- 温度设 0 不保证格式合规，别指望；
- 流式输出拿到的是半截 JSON，要么缓冲到结束，要么用增量解析器；
- 修复替换可能误伤字符串内部——所以严格解析永远排第一，修复只做最后兜底；
- 日志别打全量原始输出，截断 + 脱敏。

## 可复用建议

- `extractJSON`/`parseLoose` 做成纯函数工具，插件间复用，别每处内联一份；
- 线上每次解析失败的真实样本收进 fixture 变成单测——这个文件就是你的回归资产；
- 顶层约定"单个 JSON 对象"而非数组：数组易被截断，对象还能塞 `version` 字段做兼容；
- 运行时支持结构化输出/工具调用 JSON 模式就开，但防御层照样保留——不同模型、不同网关行为不一致；
- prompt 层约束（"只输出 JSON，无围栏"）照写，但只当第一道纱窗，不当承重墙。

## 总结

对 LLM 输出做解析，本质是对接一个"大部分守约、偶尔抽风"的下游。分层的意义在于每层只做一件小事：提取解决格式混杂，修复解决低级语病，校验解决结构正确，重试解决偶发失败。五层都很便宜，但任何一层单独扛，都会在某个深夜的批处理里给你惊喜。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/25180ac7b5960fb6.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/24b896e122eadaab.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/1cf771a7e1761252.png)

