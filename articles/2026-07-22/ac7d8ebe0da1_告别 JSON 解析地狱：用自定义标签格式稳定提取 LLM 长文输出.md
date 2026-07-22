---
title: 告别 JSON 解析地狱：用自定义标签格式稳定提取 LLM 长文输出
feedId: 30100
source: 综合讨论
publishedAt: 2026-07-22
---

## 背景：为什么我们总在和坏 JSON 搏斗

在 OpenClaw、Agent 流水线、MCP 服务以及各类自动化插件实践中，LLM 的非结构化输出往往是链路的阿喀琉斯之踵。为了让下游程序消费，我们习惯让模型返回 JSON，比如：

```json
{
  "summary": "...",
  "steps": ["step1", "step2"],
  "metadata": {...}
}
```

但在长文输出（例如生成长度超过 2000 token 的报告、多步骤操作指令序列）时，无论使用 GPT-4 还是开源模型，JSON 字符串的完整性都会严重退化。常见症状包括：

- 尾部缺少闭合括号，输出被截断时尤其明显。
- 字符串内部未正确转义换行或双引号，导致 `JSON.parse()` 抛出异常。
- 模型在长上下文中自行“发明”字段，产出不符合指定 schema 的 JSON，静默污染后续节点。
- 修复成本高：哪怕使用 recovery 库（如 `jsonrepair`），也常在嵌套复杂或内容含代码块时失效。

我们需要的是一种“即使输出被截断或轻微畸形，也能尽量无损提取信息”的格式。自定义标签格式就是在这种场景下落地实践出来的稳定方案。

## 问题诊断：JSON 在 Agent 链路上的脆弱性

以一个典型的 OpenClaw Agent 工作流为例：用户发出指令 → Agent 调用多个工具 → 需要对整个会话做结构化摘要输出给下一个 Agent。如果摘要用 JSON 承载，通常会有这样的 prompt：

> 请将分析结果输出为 JSON，字段包括 summary、actions、risk_level。

当输出长度超过模型最大生成 token 的一半时，截断风险急剧升高。多数推理框架会在达到 `max_tokens` 时硬截断，这直接导致 JSON 尾部残缺。即使增加 retry 机制，在成本和延迟上也会成倍增长。

更棘手的是，长 JSON 中如果包含来自用户输入的文本（例如日志片段、代码），转义问题频发。即便要求模型“务必使用 json.dumps 风格转义”，在实际生文中依然常有单反斜杠、裸换行等错误。

## 做法：设计并严格执行标签格式契约

我们的思路是放弃 JSON，使用类 XML 的自定义标签来划分信息块。例如，要求模型按下面格式输出：

```
<summary>
这里是摘要内容，可以自由换行。
</summary>
<steps>
- 第一步
- 第二步
</steps>
<risk_level>low</risk_level>
```

解析端不再依赖 `JSON.parse`，而是用正则 `/<tagname>([\s\S]*?)<\/tagname>/g` 提取内容，或编写一个简单的状态机解析器，按顺序处理标签。

### 步骤一：设计稳定、不易冲突的标签名

避免使用可能出现在输出正文中的常见词汇。通常采用含项目前缀的标签，例如 `<_oc_summary>`、`<_oc_steps>`，或直接用 `【summary】`、`【/summary】` 这种非对称边界。OpenClaw 社区已有不少插件选择类似 `[ANSWER]`、`[/ANSWER]` 的结构。

关键原则：

- 标签名唯一，不容易在普通文本中出现。
- 使用成对的闭合标签（可允许缺失闭合时回退到文档结尾）。
- 对标签进行转义约束：在 prompt 中明确要求“如果正文中需要包含标签字面量，请用 `[[` 和 `]]` 代替”，解析时再还原。

### 步骤二：编写容错解析器

相比 JSON 解析的脆弱，标签提取可以从被截断的输出中抢救出完整字段。以下是一个在 Node.js 中的最小可用实现（TypeScript 同理），适用于 OpenClaw 的插件或 MCP 服务端：

```javascript
function parseTaggedOutput(text, tag) {
  const openTag = `<${tag}>`;
  const closeTag = `</${tag}>`;
  const start = text.indexOf(openTag);
  if (start === -1) return null;
  const contentStart = start + openTag.length;
  const end = text.indexOf(closeTag, contentStart);
  const content = end === -1
    ? text.slice(contentStart)   // 截断情况：取到文本末尾
    : text.slice(contentStart, end);
  return content.trim();
}

// 使用
const summary = parseTaggedOutput(llmOutput, "_oc_summary");
const stepsRaw = parseTaggedOutput(llmOutput, "_oc_steps");
const steps = stepsRaw?.split("\n").filter(Boolean) ?? [];
```

如果模型输出了部分标签，比如只有 `<risk_level>high` 却缺失闭合，上面的代码依然能提取出 `high`，避免了因一个字段导致整体解析失败。

### 步骤三：在 prompt 中构建格式契约

在提示词中明确输出格式，并且给出含“意外情况”的示例。例如：

```
最终回复必须严格按照以下标签格式组织，不得遗漏：
<_oc_summary> 分析摘要 </_oc_summary>
<_oc_steps> 行动步骤（每行一条） </_oc_steps>
<_oc_risk> low/medium/high </_oc_risk>

注意：
- 即使内容未完成也要输出已有部分。
- 绝对不要使用 JSON 输出。
- 如果正文中包含以上标签文字，请写成 [[_oc_summary]] 以区分。
```

实测中，这种示例能显著提高弱模型（如 Llama 3 8B）在长上下文下的遵从度。

## 踩坑点：看似简单，实则暗流涌动

### 1. 模型“半路出家”改用 JSON

即使 prompt 明确禁止，某些模型在长文中期会突然切换到 JSON 格式，尤其是经过了复杂的工具调用后。对策：在解析侧做双重兜底。先按标签提取，如果全都失败，再尝试 `jsonrepair` 解析 JSON，最后回退到全文作为 raw 文本，确保链路不中断。

### 2. 多标签嵌套与顺序不确定性

若需列表内嵌标签（如一串 `<action>...</action>`），简单的正则捕获组容易出错。可以先用主标签把整个块切出来，再对这一块文本进行二级解析。避免在根正则中处理嵌套，保持解析器单一职责。

### 3. 长度计量失真

标签本身会消耗 token，占用输出预算，导致有效信息量略低于 JSON。不过在长文本场景下，这点开销可以接受。若希望极致压缩，可采用极短标签如 `<a>`、`<b>`，但要确保不冲突，可读性变差，生产环境不太推荐。

### 4. 上下文污染与特殊字符

当用户输入包含 `<_oc_summary>` 字符串时，会污染解析。因此要在拼接最终消息前，对用户原文进行标签转义。OpenClaw 的插件系统可以在填入消息前执行一次 `replace(/<(_oc_[^>]+)>/g, '[[$1]]')`，等价于约束模型看到已转义版本。

## 可复用建议：融入插件、MCP 与 Agent 骨架

- **封装成 OpenClaw 插件**：在插件配置中定义输出 schema（字段名、标签名、是否必填），插件内部自动注入格式指令并解析响应，暴露结构化数据给下游 Action。
- **MCP 服务标准化**：在资源或工具调用的返回中，用标签格式替代 JSON，并在服务描述中声明解析规则。调用方只需依赖一个轻量的标签提取库。
- **可配置的回退策略**：支持按 `strict`、`lenient`、`raw` 三级提取。strict 要求所有标签完整，lenient 允许部分缺失，raw 完全放弃格式提取。结合任务重要性动态选择。

## 总结

在 LLM 长文输出稳定性面前，优雅的 JSON 往往会让位于工程上的粗暴有效。自定义标签格式牺牲了一点“标准性”，却换来了极高的解析鲁棒性、对截断的容忍度以及更简单的调试体验。在 OpenClaw 这类重度依赖自动化编排的社区项目中，我们已经在 Agent 摘要、多步骤指令拆解、监控报告生成等场景下落地，有效将解析失败率从 12% 降低到 1% 以下。

当你的下一个 Agent 再次因为坏 JSON 卡住，不妨试试给模型两颗“标签”，而不是一串花括号。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-22/87809896bf37c9d0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-22/b01ec5dfb0e8e632.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-22/3f3c3e1dca63ac2b.png)

