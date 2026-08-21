---
title: OpenClaw 实战：subagent 并行编排，别再单线程跑复杂任务
feedId: 34094
source: 综合讨论
publishedAt: 2026-08-22
---

## 背景

在 OpenClaw 里跑复杂自动化时，单个 agent 经常被不同性质的子任务拖慢。比如一次发布前检查，既要从 GitHub 拉变更列表，又要跑 lint，还要检查文档链接。如果全塞给一个 agent，它会反复切换工具，上下文里堆满中间输出，最后判断质量下降。更实际的是，这些步骤大多互不依赖，完全可以并行。

subagent 编排适合的场景：

- 只读检索、扫描、检查类任务多
- 每个子任务输入输出边界清晰
- 结果可以结构化汇总

不适合：强依赖前序结果、需要共享大量隐式上下文、多个子任务写同一路径。

## 做法

### 1. 拆出独立 subagent

以“发布前检查”为例，我拆成三个：

- `change-summarizer`：只读 PR diff 和 commit
- `lint-runner`：只跑 lint 命令并返回结果
- `doc-link-checker`：只查文档外链

每个 subagent 都有自己的 system prompt 和工具白名单，不共享主 agent 的全部工具。

### 2. 通过 MCP 控制权限

OpenClaw 里我会把 GitHub、文件系统、执行命令等能力以 MCP server 形式接入，但在每个 subagent 配置里只暴露必要工具。例如 `lint-runner` 只有 `run_command` 的受限版本，且只能跑 `npm run lint`，不能写文件。

### 3. 定义统一返回协议

主 agent 下发给每个 subagent 的输入包含：任务目标、仓库路径、检查范围、输出格式。输出统一为 JSON：

```json
{
  "status": "success|failed",
  "summary": "一句到三句话",
  "files": ["path"],
  "errors": []
}
```

不要让 subagent 把原始日志直接作为 summary 返回。

### 4. 并行触发并设置超时

主 agent 同时调用三个 subagent，每个带 120 秒超时。主 agent 在这期间只等待结果，不干预执行。合并时只读取 summary 和 files 字段，需要细节再打开对应文件。

### 5. 汇总与仲裁

主 agent 只拿结构化结果做最终判断。如果 lint 和 doc 检查同时报告问题，按优先级合并，不重复展开。

## 踩坑点

- **上下文膨胀**：如果 subagent 返回大段日志，主 agent 再拼接，后续推理会被噪声淹没。解决：原始输出全部落盘到任务结果目录，subagent 只回传路径和摘要。
- **写冲突**：两个 subagent 同时改文件会互相覆盖。默认只读，写操作全部由主 agent 汇总后执行；如果必须并行写，按目录或文件名前缀划分。
- **MCP 并发限制**：不少 MCP server 并不支持高并发，尤其浏览器、终端类工具。主 agent 侧要限制同时运行的 subagent 数量，不要一次开 10 个。
- **错误被当成事实**：subagent 失败返回的 traceback 容易被主模型误读成业务信息。协议里把 `status: failed` 和 `errors` 独立出来，主 agent 对失败任务只做标记，不解读原始错误。
- **成本与延迟**：并行不一定更快，如果 MCP 服务本身是瓶颈，开 5 个 subagent 只会排队。先测单个 subagent 的耗时，再决定并发数。

## 可复用建议

- 先用串行跑通每个 subagent，验证工具权限和返回格式，再切换并行。否则多个错误叠加很难定位。
- 返回协议固定后，用 JSON Schema 或简单断言校验，减少主模型自由发挥。
- 大段共享上下文放文件，通过路径传递，不要全量塞进 prompt。
- 每个 subagent 的 system prompt 写“只能做什么”，而不是“不要做什么”，权限更可控。
- 记录每个 subagent 的耗时、token 消耗、失败率，用这些数据决定是否保留或合并 subagent。

## 总结

subagent 并行编排的收益不在“同时跑多个 AI”，而在提前划分清楚任务边界、工具权限和失败路径。OpenClaw 的 agent 配置和 MCP 工具隔离让这种模式可以落地，但主 agent 的拆解与汇总规则才是关键。对于只读多、I/O 密集的自动化任务，值得拆并行；对于强依赖和写密集流程，串行反而更稳。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/cf1150543606c2c4.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/4c100e4526c2127f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/d2355a86e306066a.png)

