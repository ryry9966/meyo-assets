---
title: Prompt 工程实战：Cron 任务应该怎么写 instruction
feedId: 31895
source: 综合讨论
publishedAt: 2026-08-06
---

# Prompt 工程实战：Cron 任务应该怎么写 instruction

## 背景
在 OpenClaw 的自动化实践中，Cron 插件是实现定时任务的核心组件。它会在预设的时间点触发 Agent，把当前时间、系统变量和一段 instruction 一起推送给推理后端。很多同学在配置定时任务时，只写一句 “总结今天的工作”“检查系统状态”，然后发现 Agent 产出的内容完全不可控：要么重复处理历史数据，要么输出一堆 Markdown 让下游无法解析，更严重的还会因为缺少边界条件跑出幻觉，把测试数据写进生产环境。

根本原因在于：Cron 任务是一次次独立的会话，Agent 的每一次执行都依赖 instruction 足够明确、结构化，才能稳定完成任务。

## 问题拆解
Cron 任务和手动对话最大的区别有两个：

1. **每次触发都是冷启动**：Agent 不会记得上一次运行的状态，除非你在 instruction 中显式提供了外部状态（如数据库里的游标、上次处理到的 id、状态标记等）。
2. **输出要被机器消费**：定时任务往往需要推送通知、更新数据库、生成报表，如果输出格式不固定，下游解析成本极高，甚至直接失败。

基于这两点，Cron instruction 写得好不好，基本决定了定时任务的生产稳定性。

## 实战步骤：写出可托管的 Cron Instruction

### 1. 用任务声明代替自然语言请求
不要写 “请你帮我看看最近有什么重要的事”。要写清晰的 Task声明，动词精确，减少歧义。

```text
Task: 汇总未处理的用户反馈，按优先级排序，生成 JSON 报告。
```

然后加上 “不做什么” 的显式约束：

```text
不要包括已关闭或已回复的反馈。不要重新生成昨天的报告。
```

### 2. 严格限定时间窗口与作用域
Cron 的本质是定时，频繁出问题的地方就是时间范围没锁死。Agent 默认可能去拿 “最近的数据”，但对它来说 “最近” 的概念并不稳定。

做法：
- 使用模板变量注入精确时间。OpenClaw 的 Cron 插件可以在 instruction 中注入 `{{ current_iso }}` 和 `{{ last_run_iso }}` 之类的占位符。
- 在 instruction 中明确写出 SQL 或过滤条件，比如：

```text
只查询 created_at 在 '{{ last_run_iso }}' 和 '{{ current_iso }}' 之间的记录。
```

如果没有 last_run，就精确给一个时间窗口，比如 “过去24小时”。

### 3. 强制 output schema
OpenClaw 的 agent 一般支持 JSON mode 或 function calling，建议 Cron 任务都约束为 JSON 输出。直接在 instruction 里写死一个 schema 往往比 “Please return JSON” 有效得多。

示例：

```text
所有响应必须是一个 JSON 对象，符合以下结构：
{
  "summary": "...",
  "alerts": [{"level": "info|warn|error", "message": "..."}],
  "processed_count": 123
}
不要输出任何前置解释。不要用 markdown code block 包裹 JSON。
```

如果后端支持，可以直接使用 function_call 的 schema 定义，这样幻觉更少。

### 4. 幂等性处理
Cron 任务一旦重复执行，如果没有幂等逻辑，可能会导致重复写入、重复通知。instruction 里必须包含幂等指令，比如：

```text
对每一条未处理的记录，在获取后立即将其状态标记为“处理中”，完成后标记为“已处理”。
若某条记录已被标记为“已处理”，则跳过。
```

如果无法直接改状态，也可以在任务最后输出已处理 id 列表，由下游做去重，但要明确说明。

### 5. 容错与截断
长列表、网络超时都可能让 Agent 半途而废。在 instruction 里要指挥 Agent 做局部降级：

```text
如果数据量过大（超过50条），只处理前50条，并在输出中包含剩余未处理数量。
如果某个单独条目处理失败（如外部 API 调用超时），记录它的 id 和错误原因到 error_log 字段，不要中断整个任务。
```

## 踩坑实录

- **默认时区不一致**  
  Agent 理解的时间可能是 UTC，而业务数据库是 UTC+8，导致时间窗口错位。务必在 instruction 中说明：“所有时间均为 Asia/Shanghai”。
- **自然语言输出打断了自动化链路**  
  早期我们偷懒没约束格式，Agent 回复 “好的，已经处理完毕”，下游的 webhook 解析器直接报错。后来全部强制 JSON 输出才稳定。
- **遗忘 dry-run 导致脏数据**  
  测试 Cron 任务时，一定要在 instruction 中加：“本执行仅为测试，所有写入操作请用 dry-run 模式，不得真实修改数据”。等验证通过后再移除。
- **没有告诉 Agent 可以调用哪些工具**  
  OpenClaw 里如果 Cron 任务的 Agent 绑定了 MCP 工具，instruction 中最好显式列一下可用工具名称和用途，减少乱调用的情况。

## 可复用的 Instruction 模板

```text
Task: {{ task_description }}
Execution time: {{ current_iso }}
Time window: {{ last_run_iso }} 至 {{ current_iso }} (时区 Asia/Shanghai)
Input source: {{ source_tool }}
Only fetch records where status = 'unprocessed' and created_at within window.
Output must be a single JSON object following the schema below:
{
  "processed": [...],
  "errors": [...],
  "remaining_unprocessed": int
}
Do not include any text outside the JSON.
If total records > 100, stop at 100 and set remaining_unprocessed accordingly.
Mark each processed record as 'done' via {{ update_tool }}.
```

根据实际任务替换模板变量即可。多次实践下来，这套模板几乎能覆盖 80% 的 Cron 自动化场景。

## 总结
Cron instruction 不是一句美好的祈使句，而是一份可执行的程序规范。每一个模糊的用词最后都会变成凌晨三点的告警。把时间窗口锁死，把输出格式焊紧，把容错路径写清楚，定时任务才能真正做到零干预运行。在 OpenClaw 的工程化实践中，花 20 分钟打磨一段 Cron instruction，比花 一晚上修复脏数据划算太多。

---

