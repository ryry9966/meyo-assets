---
title: Agent 的 subagent 编排：多个 AI 并行做事的实践与排坑
feedId: 33682
source: 综合讨论
publishedAt: 2026-08-18
---

## 背景

在 OpenClaw/Agent/MCP 的自动化实践里，单 Agent 处理长链路任务很容易撞到三个天花板：

1. **上下文爆炸**：多步骤任务把大量工具输出、中间结果都塞进同一个上下文，后期推理质量明显下降。
2. **串行太慢**：数据采集、网页解析、文件处理等步骤一个接一个跑，总耗时线性累加。
3. **工具冲突**：同一个 Agent 同时操作多个数据源或文件时，容易互相污染状态，排查困难。

subagent 编排是一种务实的解法：主 Agent 负责任务拆分与结果收敛，多个 subagent 各自领一块独立任务并行执行。它不是为了“并发”而并发，而是用边界清晰的子任务换取更稳的上下文和更快的整体吞吐。

## 问题

不是所有任务都适合拆成 subagent。强依赖、需要全局视角、步骤间共享大量中间状态的任务，拆开后反而要花更多精力做信息同步。真正适合并行的场景通常满足：

- 子任务之间依赖少；
- 每个子任务有自己的数据源或工具集合；
- 最终结果可以独立校验后合并。

如果不加约束直接开多个 subagent，常见翻车点包括：重复劳动、结果格式漂移、写操作冲突、MCP 连接数爆掉、token 成本翻倍。

## 做法/步骤

### 1. 按边界拆分，不按步骤拆分

拆分维度优先考虑数据源、工具域、文件目录或业务实体。例如：

- 收集类：subagent A 负责搜索引擎采集，subagent B 负责 API 拉取，subagent C 负责本地文件解析；
- 处理类：每个 subagent 处理一个目录或一类文档；
- 分析类：一个 subagent 做数据清洗，一个做异常检测，一个做报告生成，但前提是前两者产物已落盘。

避免把“先查 A 再根据 A 结果查 B”这种强依赖链路硬拆成并行。

### 2. 定义 subagent 契约

主 Agent 和 subagent 之间用 JSON 通信，不强依赖自然语言自由发挥。每个 subagent 至少接收：

```json
{
  "task_id": "sub-003",
  "goal": "解析 data/raw/2024 下所有 CSV 文件并提取字段",
  "context": "主任务目标是生成 Q3 报告，csv 文件可能包含日期列",
  "allowed_tools": ["read_file", "list_files"],
  "output_schema": {
    "status": "ok|error",
    "result": "数组",
    "evidence": "关键文件路径或日志"
  }
}
```

输出必须强约束 schema。主 Agent 拿到结果后先校验字段完整性，再进入汇总阶段。

### 3. 主 Agent 并行调度，限制并发

在 OpenClaw 里可以用 subagent 启动器或线程池管理并行。核心不是“开得越多越好”，而是根据工具连接数、API 限流和上下文大小限制并发数。工程上建议并发 3–5 个，先跑通小样本再放大。

伪代码示例：

```text
subtasks = planner.split(goal, boundaries)
pool = SubagentPool(max_workers=3)
results = pool.run(subtasks, timeout=180, max_tokens=4000)
validated = validator.check_schema(results)
merged = merger.deduplicate_and_merge(validated)
```

### 4. 汇总与收敛

主 Agent 做三件事：

- 冲突检测：多个 subagent 返回同一实体但数据不一致时，保留证据路径，不自动覆盖；
- 去重：按 task_id + 数据源 URL/文件 hash 去重；
- 格式统一：把不同 subagent 的输出映射到最终报告/数据库结构。

## 踩坑点

### 上下文隔离导致信息缺失

subagent 默认看不到主 Agent 的完整上下文，可能做出重复劳动或偏离主任务。对策：在 `context` 字段里附上共享事实清单，明确“不要重复采集这些 URL”“该目录已由其他 subagent 处理”。

### 写操作冲突

多个 subagent 同时写同一个文件、数据库表或缓存 key，会产生覆盖、锁等待或脏数据。对策：subagent 默认只读共享资源；所有写操作由主 Agent 串行执行，或按 namespace/租户隔离写入路径。

### MCP 连接数限制

每个 subagent 独立初始化 MCP client 时，很容易把远程 MCP server 或本地代理的连接池打满。对策：复用共享 MCP client，或者在主 Agent 层统一做工具调用，subagent 只产出“工具调用计划”而非直接调用。

### 结果格式漂移

即使给了 JSON schema，subagent 仍可能输出多余文本、缺失字段或类型错误。对策：主 Agent 做两层校验，先用 JSON parser，再用 schema validator；校验失败直接重试一次，仍失败则降级为串行处理。

### 成本与超时

并行能缩短 wall-clock time，但总 token 消耗通常比串行更高，因为每个 subagent 需要重复一部分上下文。对策：设置 `max_tokens`、单任务超时、总预算上限；对纯只读子任务使用更小的模型或限制上下文长度。

## 可复用建议

- **先串行，后并行**：串行版本跑通后再拆 subagent，这样最能暴露拆分边界问题。
- **任务幂等**：每个 subagent 的任务描述里带唯一 task_id 和幂等键，失败重试不会重复产生副作用。
- **日志追踪**：全链路带 run_id，subagent 日志落盘到 `logs/{run_id}/{task_id}.jsonl`，方便定位是哪个子任务拖慢整体。
- **保留人工闸口**：合并后的结果先落盘为草稿，不直接推送给下游系统；人工或规则确认后再写库。
- **限制 subagent 权限**：只给该任务需要的工具集，不要把主 Agent 的全套工具无差别复制过去。

## 总结

subagent 编排的收益不是“多 AI 同时干活”听起来酷，而是让每个 Agent 的上下文更短、任务更聚焦、失败更容易定位。真正能稳定跑起来的方案，核心永远是三件事：**边界清晰、契约明确、收敛机制**。没有这些，并行只会把问题放大成更多并行的错误。

在 OpenClaw 生态里落地时，优先复用已有的 MCP 工具和日志体系，避免为了 subagent 引入新的服务依赖。先从 3 个以内的小并行开始验证，比一次性上 10 个并发更接近工程现实。

---

