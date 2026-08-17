---
title: Subagent 编排实战：让多个 AI 并行处理，而不是在主上下文里堆任务
feedId: 33616
source: 综合讨论
publishedAt: 2026-08-17
---

## 背景

在 OpenClaw 里做自动化时，很容易形成一种“主 Agent 包办一切”的习惯：查资料、跑脚本、看日志、做总结，全部塞进一个会话。任务一多，问题就开始出现：上下文被大量中间结果污染；多个任务串行执行，耗时长；某个工具调用报错，整个流程卡住；最后汇总时模型开始出现“选择性遗忘”。

后来我把任务拆给多个 subagent 并行处理，主 Agent 只做拆解、验收和汇总。这个做法不算新，但在 OpenClaw/MCP/插件场景下，工程化细节决定了它能不能稳定跑起来。

## 问题

并行不是简单开多个会话。多个 AI 同时做事，主要会遇到三个问题：

1. **上下文隔离**：每个 subagent 必须有独立上下文，不能互相看到对方的中间输出。
2. **任务分配**：不能让多个 subagent 抢同一个任务，也不能让任务漏掉。
3. **结果契约**：如果 subagent 回复格式自由发挥，主 Agent 汇总时基本没法可靠解析。

下面这套做法是我在一类“代码库健康检查”自动化里跑通的：主 Agent 把任务拆成依赖漏洞检查、README 一致性、测试覆盖率三个独立任务，交给 subagent 并行执行。

## 做法/步骤

### 1. 任务拆解与入队

主 Agent 不直接调用 subagent，而是先产出一个任务清单。每个任务包含：

```json
{
  "id": "dep-check-001",
  "type": "dependency_scan",
  "payload": { "repo": "repo-name" },
  "status": "pending",
  "result_schema": {
    "findings": "array",
    "risk_level": "string",
    "evidence": "string"
  }
}
```

任务进入 SQLite 或 Redis 队列。我用 SQLite 多，因为 OpenClaw 场景通常单机部署，少一个外部依赖。

### 2. 用 MCP 暴露任务队列

这是关键一步。主 Agent 和 subagent 之间不要直接传长文本，而是通过 MCP server 暴露三个工具：

- `claim_task(type)`：认领一个 pending 任务，状态改为 claimed。
- `report_result(task_id, result)`：按预设 schema 提交结果。
- `fail_task(task_id, reason)`：标记失败并记录原因。

这样做的另一个好处是：以后换成不同模型、不同 subagent 实现，接口不变。

### 3. 启动 subagent worker

每个 subagent 是一个独立会话或进程，循环执行：

1. 调用 `claim_task` 拿任务；
2. 根据 `payload` 调用自己的插件或工具；
3. 按 `result_schema` 组装 JSON；
4. 调用 `report_result` 回写。

并发数控制在 3-5 个。模型用便宜的版本跑 worker，主 Agent 用强模型做验收。

### 4. 主 Agent 汇总

主 Agent 等待所有任务进入 done 或 failed。对 done 任务做 schema 校验，对 failed 任务决定是否重试。最终生成一份健康检查报告。

## 踩坑点

### 上下文漂移

subagent 经常会把简单任务“想复杂”。比如让它查依赖漏洞，它开始分析代码风格，最后返回一大段自然语言。解决方法是在 `result_schema` 里强制要求 JSON 输出，并在 prompt 里写明“只做这件事，不要附加分析”。不满足 schema 的结果直接按失败处理。

### 并行写冲突

两个 subagent 同时写同一个文件，或者同时调用一个有状态的服务。稳定的做法是按资源分片：一个 subagent 只负责一个目录、一个服务或一类检查。如果必须共享资源，用 append-only 日志或文件锁。

### 假并行

开 8 个 subagent 不一定比 3 个快。瓶颈通常在主 Agent 的拆解速度、工具调用延迟和模型限速。先小规模压测，找到吞吐拐点再放大。

### 失败静默

最常见的问题：某个 subagent 超时或异常退出，没有回写状态，主 Agent 一直等。这需要租约机制：`claim_task` 时记录 lease 时间，超过 N 秒未 done 的 claimed 任务自动重新变回 pending。

## 可复用建议

- **任务状态机保持简单**：pending → claimed → done/failed，不要引入复杂状态。
- **输入输出契约比 prompt 更重要**：写好 `result_schema` 和 `payload`，比给 subagent 写一大段指令更可控。
- **worker 能力收窄**：单个 subagent 只做查资料、跑测试、代码检查等一件事，不要让它“顺便看看”。
- **做好可观测性**：记录每个 subagent 的 claim 时间、耗时、工具调用次数和失败原因。没有这些数据，并行问题的排查基本靠猜。
- **主 Agent 只验收，不参与执行**：一旦主 Agent 开始替 subagent 做具体工作，隔离和并行就失去了意义。

## 总结

Subagent 编排的核心不是“多开几个 AI”，而是把任务当作资源来管理：独立上下文、明确契约、可控并行、可重试。OpenClaw 的 MCP 能力让这套做法可以比较自然地落地。跑稳之后，主 Agent 的上下文会明显干净，自动化流程也更敢放大规模。但前提是把队列、schema、超时这些“无聊”的工程细节先做好。

---

