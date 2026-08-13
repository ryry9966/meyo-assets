---
title: OpenClaw 实战：用 subagent 并行扫描 monorepo 的编排与排障
feedId: 33008
source: 综合讨论
publishedAt: 2026-08-13
---

## 背景

最近在维护一个包含 14 个服务模块的 monorepo，每季度要做一次依赖健康度扫描。单靠一个 Agent 串行跑，要反复切换目录、读取 package.json、执行 `npm audit`、整理结果，耗时且容易在中途丢失上下文。后来我把这个过程改成“一个主 Agent 调度 + 多个 subagent 并行扫描”，速度和稳定性都有明显提升。

这里分享的不是“多开几个 AI 聊天窗口”，而是在 OpenClaw/Agent 工具链里，实际可复现的 subagent 编排方式。

## 问题

串行扫描有四个典型问题：

1. **上下文污染**：一个对话里同时处理多个模块，工具输出互相干扰。
2. **慢**：14 个模块逐个 `npm audit`，加上网络等待，整体耗时接近线性叠加。
3. **失败难隔离**：某个模块扫描报错，后面的模块容易被带偏。
4. **汇总困难**：自由文本输出很难统一格式，最后整理成报告非常痛苦。

## 做法

我采用“orchestrator + worker”模式，主 Agent 只做任务拆分和结果汇总，具体扫描由多个 subagent 并行完成。

### 1. 定义主 orchestrator

主 Agent 的 system prompt 里明确禁止直接执行扫描，只允许三件事：生成任务清单、下发 subagent、校验合并结果。

比如：

```text
你是 monorepo 健康度扫描的调度器。
你不得直接读取源码或执行 audit 命令。
你的职责：
1. 根据模块列表生成 task manifest；
2. 将任务分发给对应 subagent；
3. 收集结果，校验格式，合并为总报告。
```

### 2. 定义 worker subagent

在 OpenClaw 中为每个 worker 挂载最少权限的 MCP 工具。例如：

- `filesystem`：只读访问指定模块目录
- `shell`：仅允许 `npm audit --json`、`pip-audit` 等审计命令
- `git`：获取模块最近提交信息

每个 worker 只会收到自己负责的模块路径和检查项。不要给它全局 shell 或写权限。

### 3. 生成 task manifest

主 Agent 先列出模块清单，生成类似下面的结构化任务：

```json
[
  {
    "module": "billing-service",
    "path": "./services/billing-service",
    "checks": ["npm-audit", "license-scan", "git-last-commit"]
  },
  {
    "module": "user-service",
    "path": "./services/user-service",
    "checks": ["npm-audit", "license-scan", "git-last-commit"]
  }
]
```

### 4. 并行下发与执行

主 Agent 根据 manifest 并行启动多个 subagent。并发数控制在 3～5 个，避免触发 npm registry 限流或本地 CPU 跑满。

每个 subagent 的返回格式被约束为：

```json
{
  "module": "billing-service",
  "status": "ok",
  "findings": [],
  "errors": []
}
```

### 5. 汇总校验

主 Agent 收到所有结果后，先校验 JSON schema，失败项单独重试一次。最后合并成 `health-report.md`，同时保留每个模块的原始 JSON 输出，方便排障。

## 踩坑点

**1. 上下文隔离不够干净**

早期所有 worker 共用同一个 conversation context，结果一个 subagent 读到的文件路径会泄漏到另一个 worker。后来强制要求每个 worker 只接收任务参数，不读主 Agent 之外的全局上下文，问题才消失。

**2. shell 权限过大**

一次 worker 误执行了 `rm -rf`，虽然没有造成实际损失，但让我意识到 MCP 工具必须做命令白名单。只允许审计相关命令，其余全部拒绝。

**3. 输出格式漂移**

有些 subagent 会返回 Markdown 表格，有些返回纯文本，导致汇总失败。解决办法是在 system prompt 里给出一行示例 JSON，并让主 Agent 用简单解析器校验，格式不对直接退回重跑。

**4. 某个 worker 卡死**

MCP server 偶尔会挂起，不带超时的 subagent 会一直等。后来为每个 subagent 设置最大执行轮数和总超时，超时后标记为 `timeout`，不阻塞整体流程。

**5. token 消耗翻倍**

每个 subagent 都会装载工具说明和 system prompt，token 消耗比单 Agent 大得多。后来把 worker prompt 压缩到很短，只保留任务格式和输出约束，能省不少成本。

## 可复用建议

- **先串行跑通一个模块**，再开启并行。不要一开始就上 14 个 worker。
- **任务清单 + 结果清单**：主 Agent 只读结果清单，不要读取每个 subagent 的完整日志，避免上下文爆炸。
- **统一输出 schema**：至少包含 `status`、`findings`、`errors` 三个字段。
- **原始输出落盘**：保留每个 subagent 的原始 JSON 文件，汇总出错时不用全部重跑。
- **权限最小化**：只挂必要 MCP 工具，shell 必须白名单。
- **设置超时与重试**：每个 worker 最多重试一次，避免整体卡住。

## 总结

subagent 编排的核心不是“让多个 AI 同时聊天”，而是把可并行的独立任务拆开，用主 Agent 做调度、校验和汇总。它适合 IO 密集、工具调用多、模块间相互独立的场景，比如批量扫描、多数据源汇总、并行生成文档。不适合强依赖、需要全局一致上下文的任务。

在 OpenClaw 里，只要把 orchestrator 和 worker 的职责切干净，控制好并发和权限，subagent 并行并不复杂，而且工程收益很明显。

---

