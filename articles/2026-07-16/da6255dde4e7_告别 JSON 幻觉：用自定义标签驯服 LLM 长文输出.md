---
title: 告别 JSON 幻觉：用自定义标签驯服 LLM 长文输出
feedId: 29334
source: 综合讨论
publishedAt: 2026-07-16
---

## 背景：当结构化遇上长文本，JSON 开始失控

在 OpenClaw 这类 Agent 编排框架中，我们经常让 LLM 输出结构化的长内容——比如自动生成多章节报告、提取长文档的摘要链、或者为下游 MCP 工具准备复杂参数。初期大家自然会选择 JSON，因为“结构化数据就应该用 JSON”。

但一旦输出超过几百字，JSON 就暴露出严重的脆弱性：
- 长字符串里出现未转义的双引号、换行符，整个 JSON 直接断裂；
- 模型生成到一半“忘记”闭合花括号，重试成本飙升；
- 多层级嵌套下，漏掉一个逗号就让 `json.loads` 直接抛异常；
- 使用 `grammar` 或 constrained decoding 可以缓解，但那些方案往往绑定特定推理引擎，对普通 API 用户不够通用。

更致命的是，这种不可靠会传导到整个自动化流水线：一次失败的解析可能导致重试、降级甚至流程中断。在需要批量处理长文的场景（例如每日生成数十份行业分析稿），单靠 prompt 吊命式的 JSON 修复已经不够工程化了。

## 问题定义：我们需要“宽松的结构化”

思考本质，我们要的不是严格的 JSON 语法树，而是**能够可靠提取内容块的机器可读格式**。这个格式需要满足：
- 对 LLM 长输出友好，允许内部出现任意文本而不会破坏结构；
- 解析容错性强，即使模型输出不完整也能尽量恢复有用信息；
- 可组合、可嵌套，但不要引入过于复杂的转义规则。

于是我们转向了一种“类 XML 标签”的轻量标记格式。它不是要替代 JSON，而是在**长文本结构化提取**这个高故障率环节上做降级设计。

## 做法：设计、Prompt 与解析三步走

### 1. 标签设计原则

根据任务定义清晰的标签名，避免与 Markdown 语法冲突。例如，一个“长文摘要+关键段落提取”任务，我们可以设计如下输出格式：

```
<analysis>
整体分析内容，可以包含任意字符，包括 "双引号" 和换行。
</analysis>

<key_passages>
<passage>
第一条关键原文...
</passage>
<passage>
第二条关键原文...
</passage>
</key_passages>
```

命名上使用下划线或短横线连接，保证一眼就知道是自定义标签（如 `<user_summary>`、`<action_step>`）。注意不要使用 HTML 保留标签名，避免未来和其他工具产生歧义。

### 2. Prompt 模板

Prompt 里要明确给出格式规范，并且提供 1~2 个 Few-shot 示例。示例要包含“长文本内部出现特殊字符”的情况，以此训练模型正确闭合标签。

精简后的 system prompt 片段：

```
输出必须严格遵循以下标签结构，不要添加额外解释：

<analysis>
...这里是对全文的核心分析，可以有多段...
</analysis>
<recommendations>
<item>建议一，含"双引号"</item>
<item>建议二</item>
</recommendations>

注意：标签内可以自由使用任何字符，但必须保证开始标签和结束标签一一对应。不要使用 Markdown 代码块包裹整个输出。
```

### 3. 解析实现（Python 示例）

解析器不用 XML 标准库，因为 LLM 输出可能不规范（如缺少闭合标签）。我们采用**栈式状态机**，容忍部分错误并尽量恢复：

```python
import re

def parse_custom_tags(text: str) -> dict:
    result = {}
    stack = []
    # 简化：匹配成对的 <tag>...</tag>
    pattern = re.compile(r'<(/?)(\w+)>')
    pos = 0
    current_key = None
    start_idx = None

    for m in pattern.finditer(text):
        is_close = m.group(1) == '/'
        tag = m.group(2)

        if not is_close:
            # 开始标签
            if current_key is not None:
                stack.append((current_key, start_idx))
            current_key = tag
            start_idx = m.end()
        else:
            if current_key == tag:
                content = text[start_idx:m.start()]
                # 处理嵌套或单层
                if stack:
                    # 嵌套：存入父标签的 dict 中
                    parent_key, _ = stack[-1]
                    result.setdefault(parent_key, []).append({tag: content.strip()})
                else:
                    result.setdefault(tag, []).append(content.strip())
                # 恢复上一层
                if stack:
                    current_key, start_idx = stack.pop()
                else:
                    current_key = None
                    start_idx = None
            else:
                # 标签不匹配，忽略当前闭合，尝试继续
                pass

    # 处理未闭合标签（容错）：把剩余内容当作该标签的文本
    if current_key and start_idx is not None:
        result.setdefault(current_key, []).append(text[start_idx:].strip())

    return result
```

这段代码并不完美，但可以稳定应付 90% 以上的真实输出，剩余情况用 `try/except` 包裹后做一次修复重试即可。

## 踩坑实录：那些容易翻车的地方

1. **模型在长序列中漏掉结束标签**  
   当输出超过 2000 token 时，模型偶尔会生成 `<analysis>` 后直接结束。解决方式：在 prompt 中要求“在开始标签后立即输出内容，并在内容结束后立即使用结束标签”，并用 Few-shot 强调配对行为。

2. **标签内部混入 Markdown 代码块**  
   比如 `<passage>` 里包含了 ` ``` `，导致后续解析误判。处理：解析前用正则清理代码块边界，或者要求模型避免在 tag 内使用反引号，改用缩进引用。

3. **多段重复标签的聚合**  
   我上面的解析代码将同标签内容存入列表，便于后续步骤遍历。实践中需要根据业务语义决定是取第一个还是合并。建议在 prompt 中明确“每个标签只出现一次，或明确允许多次出现”。

4. **容错恢复带来的“脏数据”**  
   状态机自动补全未闭合标签会引入可能无效的尾部文本。此时下游处理要做防御性判断（如长度阈值过滤），而不是假设数据干净。

## 可复用建议

- **标签格式并非万能**：对于高度结构化、层级确定的短数据（如工具调用参数），依然建议 JSON，并结合 `response_format` 或工具调用模式。标签格式专用于长文本输出的结构化提取。
- **与 OpenClaw 任务编排结合**：可以在 Agent 的 “长文分析 → 提取 → 下游动作” 流水线中，把提取步骤改成输出标签格式，然后通过自定义 MCP 工具做解析，大幅降低重试率。
- **加入结构校验与自动修复**：在解析失败时，用同一 LLM 进行一次“格式修复”提示（成本很低），或者使用简单的启发式修复（如成对标签补全）。
- **标注化标签名**：最好定义一套内部约定，例如 `<o_summary>` 表示由 OpenClaw 生成的摘要，方便全链路追踪。
- **流式解析**：如果使用 streaming API，可以在收到完整结束标签时立刻触发下游动作，提升端到端延迟。

## 总结

从 JSON 到自定义标签格式，本质上是一次“为容错而设计”的工程决策。在 LLM 长文本输出的场景里，放弃严格的语法校验，换来更稳定的内容提取，是非常务实的优化。这套方案已在我们的自动化内容生成流水线中平稳运行数月，将因格式错误导致的重试次数降低了约 80%。

如果你正在用 OpenClaw 搭建需要产出长篇结构化内容的 Agent，不妨在容易崩的环节尝试切换到标签格式，或许就能告别 JSON 焦虑。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-16/68c15aef785ecb64.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-16/772426f99f43ac26.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-16/f510a622c0d2f316.png)

