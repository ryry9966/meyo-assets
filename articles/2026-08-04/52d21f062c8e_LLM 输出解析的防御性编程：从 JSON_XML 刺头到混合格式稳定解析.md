---
title: LLM 输出解析的防御性编程：从 JSON/XML 刺头到混合格式稳定解析
feedId: 31545
source: 综合讨论
publishedAt: 2026-08-04
---

## 背景

OpenClaw 或基于 MCP/插件的 Agent 流水线运行得越深，你越会意识到一个尴尬事实：LLM 的输出描述的是"确定性"，但字面格式充满了"随机性"。

Prompt 写得再严格（"你必须仅返回 JSON"），模型在长上下文、工具切换或 system/user 提示混用标签时，仍可能返回类似下面的东西：

```json
{"action":"search","query":"OpenClaw"}
<tool_call>{"action":"read_file","path":"/tmp/x"}</tool_call>
```

第一段是合法 JSON，第二段是 JSON 包裹在 XML 标签里。如果你用 `JSON.parse(raw_output)` 直接解析，遇到第二个就是直接抛异常——下游任务可能直接崩溃，且不会留下可诊断的日志。这并非暴力 token，而是解析逻辑应该防御但没防御。

## 问题定义

混合格式通常有几种典型表现：

1. **标签包裹 JSON**：`<tool_call>{"key":"value"}</tool_call>`
2. **JSON 内嵌标签字符串**：`{"content":"<thinking>...</thinking>"}`
3. **多个 JSON 块串联**，其中夹杂 Markdown 或 XML 文字（常见于模型用 `<output>` 作为分隔符，但粘连了多余说明）
4. **标签未被闭合**：模型输出被上下文窗口截断，出现 `{"unclosed": true` 或 `<tool_call>{"a":1}`
5. **两层结构混杂**：外层是 XML，内层是 JSON，但 XML 属性里塞了一个未转义的 JSON 字符串

核心矛盾：你不能指望模型具备编译器级别的括号匹配能力，但你的代码又必须接收这种"脏数据"。

## 做法/步骤：写一个 `smart_parse` 的务实步骤

### 第一步：分层抽取不是一劳永逸，而是限定失败范围

不要写一个庞大的正则去"完美匹配"所有混合标签。写一个足够可靠的小函数即可：**先用正则抽取出所有最外层标签匹配，再对标签内的内容做 JSON 解析**。解析失败则保留原始文本用于下游诊断。

```python
def extract_candidate_blocks(text):
    # 匹配 <tool_call> ... </tool_call> 这类成对标签，允许嵌套但不贪婪回溯
    blocks = re.findall(r"<(\w+)>(.*?)</\1>", text, re.DOTALL)
    return blocks
```

### 第二步：对 JSON 块做清洗与修复

对 `{` 起始 `}` 结束的片段，直接调 `json.loads`，若失败则做三级修复：

```
1. 修复截断字符串：补全末尾未闭合字符串
2. 过滤非语法控制符号：\x00-\x1f
3. 剥离最外层多余引号（极少数模型会把 JSON 当作字符串返回）
```

### 第三步：加一个"提取字符串化 JSON"的兜底

有时模型返回的是 `tool_call = {\"action\": \"read_file\"}` 这种"赋值表达式 + 字符串化 JSON"。暴力替换即可：把 `= {` 改成 `{`，去掉开头到第一个 `{` 的内容。

```python
def find_json_in_text(text):
    start, end = text.find("{"), text.rfind("}")
    if start == -1 or end == -1 or end < start:
        return None
    return text[start:end+1]
```

### 第四步：设计"可探测的失败"

解析失败时，不要只返回 None。把原始输出 chunk 存入日志字段（`raw_output`），并记录失败原因（`timeout` / `invalid_json` / `unclosed_xml`）。这让你在跑批时能识别出模型行为漂移，而不是纯粹骂模型。

## 踩坑点

**坑 1：贪婪正则第一个坑就是回溯爆炸。**
如果你一开始写的是 `r"<tool_call>(.*)</tool_call>"` 且不加 `re.DOTALL`，文本里同时出现两个 `<tool_call>` 块时，会吞掉中间所有内容。非贪婪加分支约束比回溯性能要靠谱得多。

**坑 2：JSON 嵌在标签里时，先剥外层再 JSON 解析顺序不能反。**
如果先 `json.loads` 全体文本，标签字符串会成为非法 token，直接丢 parse error。要先用标签抽出内部内容，再进入 JSON 解析。

**坑 3：修复转义经常破坏内容本身。**
为了剔除控制字符，有时会把 `\\u003c` 转义成 `<`，然后导致后续 XML 解析时多出一个标签。建议只替换真正的控制字符（`[\x00-\x1f\x7f]`），不要动反斜杠转义序列。

**坑 4：只对返回结果做 parse，不对中间步骤做。**
很多 Agent 插件只对最终 response 做解析，但中间步骤里的 `tool_use` 调用经常混入 `<result>` 或 `<error>` 标签。建议在 pipeline 的任意 tool_use 边界都调用同一个 `smart_parse`，而不是只修最终出口。

**坑 5：validate 用 schema-less 方式。**
如果一个解析结果你只需要 `action` 和 `parameters` 两个字段，不要要求所有 JSON 都有完整 schema。用 `dict.get("action")` 而不要强制 `"action" in data`。模型会偶尔漏一个字段，你需要容忍它并填充默认值。

## 可复用建议

1. **写一个 `parse_agent_output(text) -> dict` 独立模块**，不要散落在各个 tool 回调里。OpenClaw/MCP 插件通常多入口，导致有人用 JSON.parse，有人用 XML 解析器，行为不一致。
2. **统一策略：标签优先，JSON 兜底。** 正则挖出所有 `<tag>xxx</tag>` 块，若只有一块就解析块内；若有多块且第一块不是 JSON 就尝试整段提取。这个策略覆盖 90% 以上真实输出。
3. **约定容错上限。** 连续 3 次解析失败时，不要继续纠错，直接返回一个明确失败对象，把错误抛给上层 Agent 做重试或向用户暴露诊断信息。
4. **把解析器当作"协议层"而不是"数据层"。** 它应该处理边界情况，但它不背逻辑错误的锅。坏数据无法 parse 时，保证原始输入可回溯是底线。

## 总结

混合格式解析的防御性编程，不是写一个通吃所有格式的超级解析器，而是**设计一个允许失败、失败有迹可循、修复有明确边界的容错层**。对于 OpenClaw 这类面向自动化长流程的 Agent 工具，json/XML 标签混杂出现的概率远高于单次对话实验。把解析防弹做了，并发任务崩溃数量会明显下降，排查模型的上下文漂移也会容易得多。你的流水线会感谢你这一点防御。

---
*欢迎在评论区分享你们的输出解析翻车现场，或者实用的修复片段。*

---

