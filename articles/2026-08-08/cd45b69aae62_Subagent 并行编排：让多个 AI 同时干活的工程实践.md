---
title: Subagent 并行编排：让多个 AI 同时干活的工程实践
feedId: 32103
source: 综合讨论
publishedAt: 2026-08-08
---

# Subagent 并行编排：让多个 AI 同时干活的工程实践

## 背景：为什么需要 subagent 并行

单一 Agent 面对多步骤、多依赖、多资源任务时，串行执行存在明显瓶颈。典型场景如：同时抓取多个数据源、并行调用多个独立 API、同时对不同代码仓库做静态分析。如果所有操作由一个 Agent 逐项完成，总耗时等于各步骤耗时之和，且上下文窗口会被大量中间结果污染。

OpenClaw 的插件体系天然支持多工具并发调用，但当任务复杂度上升到需要多个推理链路时，单 Agent 的模式开始吃力。这时引入 subagent 编排 —— 将复杂任务拆分成多个子任务，分配到独立的 Agent 实例并行处理，再由主 Agent 汇总结果 —— 就成为一种自然的架构选择。

本文基于 OpenClaw + MCP 的插件生态，分享一套可落地的多 subagent 并行编排模式，包含工程实现细节、踩坑记录与可复用的设计原则。

## 问题定义

假设一个实际场景：我们需要对 GitHub 上三个不同的开源项目分别做 README 质量评估、最近 30 天 issue 活跃度统计、以及贡献者分布分析，最终生成一份横向对比报告。每个项目分析是独立的，涉及多次 API 调用、文本解析和总结。单 Agent 串行执行的总耗时在 40-60 秒左右，且容易因为单个项目 API 限频拖慢整体。

目标：将三个项目分析任务并行化，总耗时压缩到 15-20 秒，同时保证每个 subagent 拥有充足的上下文，不互相干扰。

## 做法与步骤

### 1. 主 Agent 拆解任务，生成 subagent 清单

主 Agent 接收用户 instruction 后，第一步不是执行具体分析，而是规划子任务。我们设计了一个 `plan_subtasks` 工具，让主 Agent 调用后返回结构化的任务列表：

```typescript
{
  tasks: [
    { id: "repo-1", repo: "openclaw/core", aspects: ["readme_quality", "issue_activity", "contributors"] },
    { id: "repo-2", repo: "mcp/servers", aspects: ["readme_quality", "issue_activity", "contributors"] },
    { id: "repo-3", repo: "anthropics/tools", aspects: ["readme_quality", "issue_activity", "contributors"] }
  ]
}
```

这个规划步骤运行在主 Agent 自身，因为需要判断任务是否可并行（子任务间无数据依赖），以及每个子任务的边界。

### 2. Subagent 的创建与并行调度

OpenClaw 的插件系统支持通过 Tool 调用 `create_subagent` 来启动一个独立 Agent。并行调度的关键在于：主 Agent 在一个推理轮次中**同时发起多个 Tool Call**。

利用 OpenClaw 的并行工具调用能力，我们让主 Agent 在同一轮次中生成了三个 `create_subagent` 调用，每个携带对应 repo 的 instruction。OpenClaw 会将这些 Tool Call 并行发送到执行引擎。

每个 subagent 的配置要点：
- 独立 system prompt：精简为“执行单一仓库分析”，避免不必要的安全说明占用 token。
- 独立工具集：只暴露 GitHub API 相关的 MCP 工具（`search_repositories`, `get_issues`, `get_contributors` 等），减少权限面。
- 超时控制：每个 subagent 设置 30 秒超时，防止个别任务挂起拖垮整体。
- 返回结构约定：要求 subagent 最终调用 `finish` 工具返回 JSON 格式结果，主 Agent 不解析自由文本，降低歧义。

伪代码流程：

```
主 Agent 调用 plan_subtasks → 得到 task_list
主 Agent 同时调用三个 create_subagent(task_1), create_subagent(task_2), create_subagent(task_3)
等待全部返回 or 超时
主 Agent 调用 aggregate_results 汇总 → 生成最终报告
```

### 3. Subagent 内部结构

单个 subagent 实际上是一个极简 Agent，拥有有限轮次的推理。我们限制其最大工具调用次数为 8 次，避免进入无限循环。其工作流为：

1. 调用 `get_readme` 获取 README 内容，分析质量
2. 调用 `list_issues(repo, state=all, since=30d)` 获取 issue 列表，统计数量、平均响应时间
3. 调用 `get_contributors` 统计贡献者分布
4. 汇总为约定的 JSON 结构，调用 `finish(result)`

如果某个 API 调用失败（如触发限频），subagent 会重试一次，重试失败则将错误信息写入返回结果的 `error` 字段，而不是直接崩溃。

### 4. 结果聚合与降级处理

主 Agent 收集到所有 subagent 返回后，执行 `aggregate_results`。这里会遇到工程质量的核心挑战：部分成功、部分失败、部分超时。我们设计的策略是：

- 所有返回按 ID 索引
- 正常结果的 `data` 直接使用
- 超时的任务（超过 30 秒未返回）标记为 `timeout`，聚合时跳过该仓库或使用默认占位符
- 带 `error` 字段的结果在最终报告中注明数据缺失原因

主 Agent 本身不重复请求，避免能耗飙升。

## 踩坑点

**坑 1：Token 窗口爆炸。** 即使每个 subagent 独立拥有上下文，但如果主 Agent 把三个 subagent 的全部原始输出都灌入汇总 prompt，token 消耗会线性增加。解决方式：`finish` 返回的数据尽量是结构化的摘要，而非大段文本；必要时在 subagent 内部做压缩。

**坑 2：并行下的 API 限频。** 当多个 subagent 同时调用同一个 GitHub API 时，很容易触发二级限频。需要在 MCP 插件层做全局的速率限制，我们给 MCP server 增加了全局 token bucket，每秒最多 5 个请求，超出则排队。这样虽然并行度受一定影响，但避免了大量 403。

**坑 3：Subagent 幻觉的叠加。** 每个 subagent 可能产生微小幻觉（如 issue 数量略微偏差），三个汇总后偏差会被放大。缓解方法：要求 subagent 在 JSON 中附带关键数据的 API 原始引用或计数依据，主 Agent 在汇总前抽样校验 1-2 个数据点。

**坑 4：超时后僵尸进程。** 早期实现中，超时的 subagent 仍在后台运行，持续消耗 API 配额。必须在超时后调用 abort 方法清理上下文并释放资源。我们在调度器中增加了 `AbortController` 绑定，确保主子任务生命周期一致。

**坑 5：死锁。** 如果主 Agent 错误地让 subagent 等待另一个 subagent 的结果，就会死锁。必须从设计上确保每个 subagent 的 instruction 是完全自包含的，task 规划阶段就避免交叉依赖。

## 可复用建议

1. **规划与执行分离。** 主 Agent 仅负责分解任务和汇总结果，不参与具体业务逻辑。这样边界清晰，调试时容易定位是规划错还是执行错。
2. **Subagent 标准化接口。** 所有 subagent 必须通过相同结构（如 JSON schema）返回结果，降低主 Agent 的解析复杂度。我们在团队内维护了一份 subagent 返回规范模板。
3. **资源保护前置。** 并行度不是越高越好。根据下游 API 的承载能力设置最大并行子任务数（我们通常设为 3-5），超过该数量则改为分批并行。
4. **可观测性。** 为每个 subagent 分配唯一的 trace ID，上传到监控系统。出问题时能快速定位是哪个仓库分析卡住，而不是黑盒等待。
5. **Fallback 路径。** 如果并行任务整体失败率超过阈值（如 50%），主 Agent 应退化为串行执行，确保仍有可用结果。这条路径必须在 prompt 中显式声明。

## 总结

Subagent 并行编排的核心不在于“同时跑多个 AI”，而在于**任务边界的明晰切分、结果的强结构约定、以及健壮的降级机制**。在 OpenClaw + MCP 的技术栈下，这套模式已经成熟到可以稳定运行在生产环境中。我们实际测试中，三仓库分析任务从平均 48 秒下降到 16 秒，且由于各 subagent 独立上下文，每个分析的质量反而有所提升。

对于需要频繁处理多源数据聚合、批量内容审核、并行代码审查等场景的团队，这是一个高性价比的工程投资。下一步可以考虑引入动态负载均衡和基于队列的 subagent 池，进一步提升资源利用率。

---

