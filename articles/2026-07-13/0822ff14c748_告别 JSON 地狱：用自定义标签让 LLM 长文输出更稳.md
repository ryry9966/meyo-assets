---
title: 告别 JSON 地狱：用自定义标签让 LLM 长文输出更稳
feedId: 28940
source: 综合讨论
publishedAt: 2026-07-13
---

## 背景：JSON 输出在 Agent 管道里的“不可靠”

在 OpenClaw 这类多步骤 Agent 中，一个常见模式是“LLM 思考 → 结构化输出 → 下游工具/插件消费”。为了让模型返回的数据可被程序解析，我们通常会要求它输出 JSON。例如：

```json
{
  "thought": "先检查文件是否存在...",
  "action": "read_file",
  "path": "/etc/config.yaml"
}
```

在短文本或简单任务里，模型正确输出 JSON 的概率很高。但一旦涉及**长文本生成**（全文摘要、多文件分析、代码重构建议），JSON 里的字符串内容就会急剧膨胀。这时状况频出：嵌套引号被错误转义、多行字符串忘记换行处理、模型在 JSON 外面多输出一段解释性文字，或者因为输出长度限制导致 JSON 被截断。这些问题在自动化管道里会直接引发 `JSONDecodeError`，让整个流程中断。

常见的补救手段——增加“只输出 JSON”的强调、使用 JSON mode、采用函数调用——的确能提升成功率，但在长文场景下仍有一成左右的翻车率。对于需要稳定运行的生产管线，这个失败率不可接受。

## 思路转变：从严格语法到弱结构化标签

如果 JSON 不可靠，我们能不能使用一种**语法容错性更强**的格式？答案是自定义标签格式。

标签格式类似于 XML，但对关闭标签、转义的要求可以自定义，解析也完全由我们控制。典型的样子如下：

```
<THOUGHT>
文件 config.yaml 存在，需要先备份。
备份命令：cp /etc/config.yaml /etc/config.yaml.bak
</THOUGHT>
<ACTION>
run_shell
</ACTION>
<COMMAND>
cp /etc/config.yaml /etc/config.yaml.bak && echo "done"
</COMMAND>
```

这种做法有几个明显好处：

- **容错**：不要求精确的逗号、引号、括号配对。字符串内容可以包含任意字符，只要不出现标签分隔符。
- **抗截断**：即使输出被中途截断，我们也能读到部分标签内容，错误处理比 JSON 优雅得多。
- **易读**：对人类和 LLM 都更友好，prompt 示例成本低。
- **可扩展**：增加新字段只需定义一个新标签，不用调整整体结构。

## 工程化做法

### 1. 设计标签 schema

优先使用**全大写英文单词加下划线**，避免与正文冲突。比如 `THOUGHT`, `FINAL_ANSWER`, `EVIDENCE_LIST`, `COUNTER_ARGUMENT`。如果担心重名，加上项目前缀，例如 `OC_THOUGHT`。

在一个典型的长文档摘要 Agent 里，我用的 schema 如下：

- `<SUMMARY>` … `</SUMMARY>`
- `<KEY_POINTS>` … `</KEY_POINTS>`
- `<ACTION_ITEMS>` … `</ACTION_ITEMS>`

每个标签内容允许包含换行、标点，但禁止再嵌套同名标签（如需嵌套，可用编号后缀，如 `STEP_1`）。

### 2. 构造 Prompt

告诉模型“必须严格按照标签格式输出，不要额外添加解释”。并给出 1-2 个完整示例。示例要覆盖长文本情况，这样模型更容易学到边界。

> **提示词片段（节选）：**
> 你的回答必须包含 <SUMMARY> 和 <KEY_POINTS> 两部分，格式如下：
> ```
> <SUMMARY>
> （此处为完整摘要，可多行）
> </SUMMARY>
> <KEY_POINTS>
> - 要点1
> - 要点2
> </KEY_POINTS>
> ```

### 3. 编写解析器

用 Python 的正则或简单状态机处理。正则方案适用于标签无嵌套、内容大概率不包含标签符号的场景。

```python
import re

def parse_tags(text: str, tag: str) -> str | None:
    pattern = rf"<{tag}>(.*?)</{tag}>"
    m = re.search(pattern, text, re.DOTALL)
    return m.group(1).strip() if m else None
```

如果存在多组同名标签，用 `re.finditer` 循环提取。需要特别注意 **re.DOTALL** 标志，让 `.` 匹配换行，否则多行内容会失败。

更鲁棒的方案是采用**栈式解析**，遇到 `<TAG>` 入栈，遇到 `</TAG>` 出栈并存储内容。这样能正确处理内容中偶尔出现的标签符号（只要不形成合法的开闭对）。在 OpenClaw 的某些插件中，我们甚至会对内容进行 HTML 实体转义的反向处理，例如允许 LLM 写 `&lt;` 来表示小于号，但这会提高复杂度，一般场景用不上。

## 踩坑点与应对

1. **模型在标签外部多写文字**  
   即使 prompt 强调“只输出标签内容”，长文总结后模型有时会多写一句“下面是你的摘要”。解析时先用 `re.search` 定位第一个 `<` 和最后一个 `>` 作为候选范围，或干脆在提取前执行 `text = text[text.find('<'):text.rfind('>')+1]` 裁剪。

2. **内容中误含标签分隔符**  
   比如用户文档里正好有 `<SUMMARY>` 字样。可以在 prompt 中让模型遇到这种情况时替换为 `[SUMMARY]`，或者使用罕见的分隔串：`<!-- SUMMARY_START -->` … `<!-- SUMMARY_END -->`。实测用注释式标签（`<!-- TAG -->`）可大幅降低冲突概率。

3. **截断处理**  
   当输出因 token 限制而中断时，可能缺少结束标签。此时可以让解析器回退到最后一个完整的开闭对，丢弃不完整片段，并在日志中告警，触发重试。

4. **多标签顺序混乱**  
   有些模型会调换 `SUMMARY` 和 `KEY_POINTS` 的顺序。解析时可按标签名独立提取，不依赖固定顺序。

## 可复用建议

- **优先用在“长文本生成 + 需下游解析”的场景**，例如报告生成、多步推理、多文件审查。简单短应答继续用 JSON 无妨。
- **标签命名带上项目前缀**，避免被模型当做示例而输出。
- **先实现正则提取，出错记录日志**。在生产中监控提取失败率，再决定是否升级为状态机解析。
- **结合重试策略**：如果标签解析失败，可将原始输出包装并重新请求模型修正，比 JSON 修正容易得多。

## 总结

自定义标签格式并不是要取代 JSON，而是在**长文结构输出**场景下的一种“降级容错”手段。它牺牲了 schema 的强约束和生态工具支持，换来了解析的极高鲁棒性。在 OpenClaw 这种注重自动化管线可靠性的框架里，这种务实的取舍往往比追求理论优雅更管用。

下次你的 Agent 在长文输出里 json.loads 失败时，不妨试试用 `<TAG>` 包一层。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-13/c4f3e7edf6ce6406.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-13/d892d657b38566aa.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-13/4bdc0b45b2581677.png)

