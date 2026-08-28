---
title: Subagent 编排实践：在 OpenClaw 里让多个 Agent 并行干活的工程化拆解
feedId: 35045
source: 综合讨论
publishedAt: 2026-08-28
---

## 背景

最近在 OpenClaw 里处理一批插件维护任务，需要同时检查多个插件的 manifest、生成 changelog、更新 README。单个 Agent 串行处理时，上下文很快被多个任务污染，改错文件、漏掉依赖时有发生。后来改成 subagent 编排，把任务拆给三个 worker 并行执行，整体时间从二十多分钟降到几分钟。

## 问题

并行编排不是简单多开几个会话。实际落地时会遇到四类问题：

- 工具/文件冲突：多个 subagent 同时写同一路径，后写覆盖先写。
- 输出格式不一致：每个 subagent 返回结构不同，主 Agent 汇总时丢失信息。
- Token 失控：某个 subagent 陷入反复搜索或读取大文件，消耗大量上下文。
- 卡死与超时：单个 worker 循环不退出，拖垮整个流程。

## 做法/步骤

1. **先拆任务**：只拆无依赖或弱依赖的单元。比如检查 manifest、生成 changelog、更新 README 可以并行，但“先读 manifest 再更新 README”这种强依赖不要拆。
2. **定义 orchestrator 与 subagent**：在 OpenClaw 中，每个 subagent 都是一个带独立 system prompt、工具白名单、max_steps 和输出格式的 worker。主 Agent 只负责拆解、下发、收集和校验，不亲自执行具体任务。
3. **并行调度**：主 Agent 一次性下发多个 subagent，并设置 timeout 和重试策略。等待时使用先完成先处理，而不是固定顺序。
4. **结果汇聚**：要求 subagent 返回 JSON 或固定结构的 markdown，主 Agent 只做 schema 校验和冲突检测，必要时抽查关键文件是否生成。

一个简化配置思路：

```yaml
subagents:
  - name: inspector
    tools: [read_file, grep]
    max_steps: 8
    output_format: json
  - name: changelogger
    tools: [read_file, write_file]
    max_steps: 6
    output_format: markdown
  - name: readme_writer
    tools: [read_file, write_file]
    max_steps: 10
    output_format: markdown
```

实际落地时，每个 worker 分配独立工作目录或文件名前缀，避免互相覆盖。

## 踩坑点

- **共享文件竞争**：即使两个 subagent 都调用 write_file，只要目标路径不同，基本不会冲突。如果必须写同一文件，就改成主 Agent 串行写入或加文件锁。
- **Token 预算不可控**：给 subagent 设置 max_steps 和单次工具返回长度限制。比如 fetch_url 只取前 8KB，搜索结果限制条数。
- **结果合并困难**：强制结构化输出是最省事的办法。例如要求 JSON 里固定有 `status`、`summary`、`files_changed` 字段，主 Agent 按字段合并，不解析自由文本。
- **卡死与超时**：主 Agent 对每个 subagent 设 timeout，超时直接标记失败，并释放资源。不要无限等待。
- **日志难以追踪**：并行任务日志交织，复现问题困难。给每个 subagent 分配 `task_id`，所有工具调用都带上这个 id，统一采集到日志文件。

## 可复用建议

- **先串行后并行**：每个 subagent 单独跑通后再开并行，避免把 prompt 问题放大成并发问题。
- **工具权限最小化**：只给当前任务必需的工具。比如 changelogger 不需要 shell，就不要给它 shell。
- **控制并发数**：通常 3-5 个 subagent 比较稳，太多容易触发限流或资源争抢。
- **设置降级路径**：某个 subagent 失败时，主 Agent 可以跳过、重试或用默认值兜底，而不是整体失败。
- **最后加一道校验**：主 Agent 不要完全信任 subagent 的返回，至少检查关键文件是否存在、字段是否齐全。

## 总结

Subagent 编排的价值在于把串行等待变成并行吞吐，但前提是任务边界清晰、输出规范、权限受控。真正的工程化收益来自约束，而不是堆更多 Agent。先在 OpenClaw 里跑通一个最小并行任务，再逐步扩展，比一开始就设计复杂编排更实际。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/b855b0cc9ee88eac.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/f6e9c4f4e35a63e3.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/20223d235f651d16.png)

