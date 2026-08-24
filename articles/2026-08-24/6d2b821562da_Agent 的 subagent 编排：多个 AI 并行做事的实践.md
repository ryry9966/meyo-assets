---
title: Agent 的 subagent 编排：多个 AI 并行做事的实践
feedId: 34519
source: 综合讨论
publishedAt: 2026-08-24
---

## 背景

在 OpenClaw 这类 Agent 环境里，单个主 Agent 经常被要求同时处理多件边界清晰但彼此独立的事：比如批量检查多个仓库的代码质量、同时读取多个数据源、或者并行生成多份报告。如果让主 Agent 顺序处理，总耗时线性叠加，而且中间结果会不断占用上下文窗口。更关键的是，所有操作共用同一套工具权限，一旦某个环节出错，排查和重试成本都很高。

## 问题

直接串行执行有几个明显的问题：

- **上下文膨胀**：主 Agent 读取大量原始内容后，后续推理容易被无关细节干扰。
- **权限混杂**：一个需要写文件的子任务和一个只读查询的子任务，如果共用主 Agent 权限，误操作风险增加。
- **失败连锁**：串行流程中前面步骤失败，后面全部阻塞。
- **调试困难**：所有步骤混在一个 session 里，难以定位是哪个子任务出了问题。

## 做法/步骤

我的实践是把子任务拆成 subagent，由主 Agent 做调度和汇总。

**1. 定义 subagent 边界**

每个 subagent 只负责一类工作，输入和输出都明确。例如：

- `repo_reader`：读取仓库代码，返回文件列表和关键结论。
- `log_inspector`：查询指定时间段的日志，返回异常摘要。
- `report_writer`：根据结构化数据生成报告草稿。

每个 subagent 的输出必须是小体积的结构化数据，例如 JSON 摘要，而不是大段原文。这样才能避免主 Agent 上下文被撑爆。

**2. 封装成工具并隔离权限**

在 OpenClaw 中，我会把每个 subagent 封装成独立 tool，或者通过 MCP server 暴露。权限按最小原则配置：只读任务的 subagent 不授予写权限，需要写操作的 subagent 限定到指定目录或沙箱环境。

示意配置：

```yaml
subagents:
  - name: repo_reader
    tools: [read_file, search_code]
    permission: read_only
    output_schema:
      files_found: []
      key_findings: []
    max_concurrency: 3
  - name: log_inspector
    tools: [query_logs]
    permission: read_only
    output_schema:
      anomalies: []
      time_range: []
    max_concurrency: 3
```

实际配置取决于 OpenClaw 的工具定义方式，但核心原则是：每个 subagent 拥有完成当前子任务所需的最小权限。

**3. 主 Agent 拆分任务并并行调度**

主 Agent 收到用户指令后，先生成任务清单。每个任务包含：

- 任务 ID
- 调用的 subagent 工具名
- 输入参数
- 期望输出格式
- 超时时间

然后用一个简单的调度器并行触发。我不会一次性拉起十几个 subagent，而是控制并发数，通常 3 到 5 个。并发过高容易触发上游 API 限流，也会让主 Agent 在收集结果时处理不过来。

**4. 汇总与校验**

所有 subagent 返回后，主 Agent 做汇总。汇总时只保留关键字段，比如状态、结论、引用位置，而不是把原始输出全部合并。如果某个子任务输出不符合 schema，主 Agent 会打回重试一次；仍然失败则标记为“需要人工处理”，不让整个流程崩溃。

## 踩坑点

- **上下文污染**：subagent 返回大段原文，主 Agent 继续分析时 token 爆炸。解决：强制结构化输出，限制每个 subagent 返回体长度。
- **权限泄漏**：给 subagent 过多工具权限，误操作影响外部系统。解决：只读优先，写操作放沙箱或二次确认。
- **并发限流**：多个 subagent 同时调用同一 API 或 MCP server，触发 429。解决：限制并发数，加重试退避。
- **结果格式不一致**：不同 subagent 输出风格不统一，汇总困难。解决：提前定义 schema，主 Agent 校验格式，不合格直接打回。
- **任务切得太碎**：调度开销大于实际执行收益。解决：每个子任务保持 3 到 10 分钟工作量，避免碎片化。

## 可复用建议

1. 先串行跑通一个完整流程，再改成并行。不要一上来就上调度器。
2. 给每个 subagent 固定角色和输出模板，减少“自由发挥”。
3. 外部服务尽量通过 MCP 封装，让 subagent 只面对统一接口。
4. 保留每次编排的 trace：任务 ID、输入、输出、耗时、重试次数。排查问题时非常有用。
5. 设置全局超时和最大重试次数，避免单个 subagent 卡死拖垮整体。

## 总结

Subagent 编排适合那些可并行、边界清晰的任务。把大任务拆成多个小任务，由主 Agent 调度多个 subagent 并行执行，能在控制上下文和权限的同时显著缩短总耗时。落地时真正重要的是任务拆分、权限隔离、输出约束和并发控制，而不是堆叠更多 Agent。对于 OpenClaw 用户来说，这可以先用自定义工具或 MCP 工具实现，跑通后再逐步优化调度策略。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/ff217b14fcb9a3d8.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/2a4f367a96eab346.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/087c2d8f3a3a3db1.png)

