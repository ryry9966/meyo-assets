---
title: OpenClaw 的 session 隔离：子 Agent 不污染主会话的工程化做法
feedId: 34837
source: 综合讨论
publishedAt: 2026-08-26
---

## 背景

OpenClaw 主会话通常承载任务目标、用户偏好、关键上下文和已授权工具。拆子 Agent 执行任务时，如果让子 Agent 直接继承并回写主 session，短期看省事，长期会把主会话越弄越“脏”：中间推理、重试日志、无关工具输出、被改过的 session 变量都会混进来。结果是 token 膨胀、主 Agent 判断被带偏，排障时也很难复现现场。

## 问题表现

- **上下文污染**：子 Agent 把长 tool output、调试信息、过程文本原样带回主会话。
- **状态污染**：子 Agent 修改环境变量、工作目录、MCP 缓存或 session 级变量。
- **权限泄漏**：子 Agent 拿到主会话全套工具权限，误触发写操作或危险调用。
- **不可回滚**：主会话混入子任务细节，重放时结果不一致。

## 做法/步骤

### 1. 先定义 I/O 契约

子任务不要返回原始日志，只返回结构化结果：

```json
{
  "ok": true,
  "summary": "生成 3 个候选配置",
  "artifacts": ["out/configs.json"],
  "metrics": {"steps": 12, "tokens": 3400}
}
```

主会话只需要 summary、artifact 路径、关键指标，不需要执行过程。

### 2. 创建独立 session，不直接继承

子任务使用独立 session 模板，不要使用默认继承模式。主会话把必要输入序列化成快照传入：

```yaml
subagent:
  session:
    inherit: false
    context_mode: snapshot
    ttl: 300
```

快照应当是纯数据副本，不共享可变引用，避免子 Agent 改到主 session 的数据。

### 3. 最小工具和 MCP 白名单

子 session 只挂完成任务必需的工具。MCP server 如果必须复用，至少启动独立实例，或者任务结束后执行 reset/close。不要直接复用主会话里的连接池，否则服务端缓存、游标、临时状态很容易串。

### 4. 隔离运行时状态

给子 Agent 独立 workdir、env prefix，避免写入 `.openclaw/state` 主状态目录。临时目录在结束时清理：

```python
run_isolated(
    task=task,
    tools=["read_file", "write_file", "mcp.search"],
    workdir="/tmp/oc-subtask-{id}",
    env={"OPENCLAW_ENV": "subtask"},
    timeout=180,
)
```

### 5. 强制输出校验

对子 Agent 返回体做 JSON Schema 校验，不合格就只回传错误摘要，不让自由文本直接进主会话。异常退出也要 finally 兜底：

```json
{"ok": false, "error": "timeout", "summary": "未完成"}
```

### 6. 超时与熔断

限制子 session 的最大步数、token、墙钟时间。超时后终止并只回传状态码，不倾倒执行历史。

## 踩坑点

- **“只读快照”不是真只读**：如果直接传原对象引用，子 Agent 仍可能改到主 session 的数据。深拷贝或序列化后再传。
- **MCP 实例复用容易串状态**：实践中遇到过子任务修改了 server 侧游标，主会话后续调用结果错位。独立实例或每次 reset 最稳。
- **白名单不能只写在 prompt 里**：必须在 runner/tool gateway 层强校验。Prompt 说“你只能用只读工具”只是建议，不是安全边界。
- **长输出不要整体回传**：大文件、长工具输出落盘后只回传路径和摘要，避免主会话上下文膨胀。
- **子 Agent 崩溃时结构丢失**：超时或异常退出要让 finally 兜底返回错误结构，否则主会话拿不到任何可解析结果。

## 可复用建议

封装一个 `run_isolated` 作为唯一子 Agent 入口，内部统一处理 session 创建、快照复制、工具白名单、输出校验、清理和错误摘要。主会话只允许通过这个入口拉取子任务结果，避免各任务各自实现导致泄漏。

隔离等级可以分级：

- `read-only`：只能读主上下文快照和指定文件。
- `sandbox`：独立 workdir/env，可写临时目录。
- `full`：显式授权才能改主会话状态，默认禁用。

最后，把子 Agent 的完整执行日志归到独立 trace，主会话不吞日志，但只保留 trace_id 和最终结果。这样主会话保持轻量，排障也能回溯。

## 总结

Session 隔离不是“完全不共享”，而是把共享控制到三个边界：输入是快照、输出是结构化结果、工具和状态有生命周期。做好这三点，子 Agent 才能成为主 Agent 的执行单元，而不是污染源。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/fea370664a3f70b6.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/6efddb82e6bc7b4c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/baf38ab3bb21c576.png)

