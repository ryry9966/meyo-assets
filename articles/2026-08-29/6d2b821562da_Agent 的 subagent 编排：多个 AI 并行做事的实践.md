---
title: Agent 的 subagent 编排：多个 AI 并行做事的实践
feedId: 35260
source: 综合讨论
publishedAt: 2026-08-29
---

## 背景

单 agent 处理复杂自动化任务时，容易同时出现两类问题：上下文被中间结果污染，以及互不依赖的任务串行排队。比如一次代码仓库巡检，要同时看依赖风险、测试覆盖、文档链接、安全漏洞。如果全部丢给一个 agent，它会在多个工具之间来回切换，最后输出冗长、难以追踪，而且耗时长。

在 OpenClaw/Agent/MCP/插件体系里，更工程化的做法是：把任务拆给多个 subagent 并行执行，主 agent 只做调度、校验和汇总。这样上下文更干净，失败可局部重试，工具权限也可以按子任务隔离。

## 问题

subagent 并行不是“开更多 AI 就会更快”。实际要解决四件事：

- 任务边界怎么切，哪些能并行，哪些必须串行。
- 子 agent 输出如何结构化回传，避免主 agent 上下文膨胀。
- 共享工作区、MCP server、API 限流如何隔离和控制。
- 单个 subagent 失败时，是否能部分成功，还是整个流程失败。

## 做法/步骤

### 1. 先拆任务

判断标准很简单：子任务之间是否共享可变状态。如果 A 的输入不需要等待 B 的输出，就可以并行。例如：

- 并行：扫描 repo A、repo B、repo C。
- 并行：同时做依赖检查、文档链接检查、API 冒烟测试。
- 串行：扫描完成后，再生成汇总报告。

### 2. 定义 subagent 模板

每个 subagent 只做一件事，并明确工具、工作目录、超时、输出格式。不要给“全能”配置。

```yaml
subagents:
  repo_scan:
    tools: [git, grep, mcp-sast]
    workdir: /tmp/run/repo_scan
    output: json
    timeout_sec: 600
    max_steps: 40
  doc_check:
    tools: [filesystem, mcp-docs]
    workdir: /tmp/run/doc_check
    output: markdown
    timeout_sec: 300
  api_smoke:
    tools: [http, mcp-openapi]
    workdir: /tmp/run/api_smoke
    output: json
    timeout_sec: 180
```

工具按域拆分，权限最小化。文档检查 subagent 不需要 git push，API 测试 subagent 不需要文件删除权限。

### 3. 主 agent 并行触发

主 agent 不要在循环里逐个等待，而应使用并行调度原语。伪代码：

```text
results = scheduler.parallel(
    repo_scan(repos),
    doc_check(doc_paths),
    api_smoke(spec_files),
    timeout=900,
    max_parallel=4
)
report = merge(results)
```

主 agent 只接收结构化摘要，不回灌完整执行轨迹。

### 4. 合并结果

要求每个 subagent 返回 JSON 或文件路径。主 agent 做去重和冲突处理。例如多个 scanner 返回同类漏洞时，先按“文件 + 规则”去重，再生成报告。不要直接把所有原始日志拼接给用户。

### 5. 失败与降级

给每个 subagent 设置超时、重试次数和部分成功策略。一个文档链接检查失败，不应阻塞整个仓库扫描结果。可以在汇总阶段标记“partial”，并输出哪些子任务未完成。

## 踩坑点

- **上下文膨胀**：subagent 的完整轨迹不要直接塞给主 agent。只回传摘要、文件路径、结构化字段。
- **共享工作区冲突**：多个 subagent 写同一目录会互相覆盖。默认给每个 subagent 独立 workdir，共享数据只读挂载。
- **限流打满**：并行 10 个 subagent 可能同时打满 API 或本地 MCP server。设 `max_parallel`，失败退避重试。
- **权限过大**：不要给文档生成 subagent 挂 git push 或 delete。按任务域挂 MCP 工具。
- **日志难追踪**：给每个 subagent 分配 run_id，日志写到 `runs/<run_id>.log`，错误信息包含 run_id。
- **格式漂移**：subagent 偶尔返回非 JSON 或字段缺失。定义输出 schema，并在主 agent 入口校验，不合法就重试或丢弃。

## 可复用建议

1. 把 subagent 定义模板化：角色、工具、输出 schema、超时、重试都做成可复制配置。
2. MCP server 按业务域拆分，避免一个 subagent 挂全部工具。
3. 主 agent 只做调度、校验、汇总，不直接执行复杂步骤。
4. 先跑 2～3 个并行小任务，验证合并与限流，再放大到 10+。
5. IO 密集、相互独立、输出可结构化的任务最适合并行；CPU/推理密集任务要评估收益。

## 总结

subagent 编排解决的不是“让 AI 更快”，而是让复杂任务可控：边界清晰、上下文隔离、失败可重试、结果可合并。实践中先拆任务，再做隔离和结构化回传，最后补上超时、限流与日志。这样多个 AI 并行做事才不会变成多个 AI 并行出错。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/af30a8e89f7f3e58.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/a35e8bd2466acbaa.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/d587fbfdbe39fc08.png)

