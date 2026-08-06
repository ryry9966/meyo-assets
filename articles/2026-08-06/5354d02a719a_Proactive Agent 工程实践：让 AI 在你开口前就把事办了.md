---
title: Proactive Agent 工程实践：让 AI 在你开口前就把事办了
feedId: 31862
source: 综合讨论
publishedAt: 2026-08-06
---

# 1. 背景

大部分 AI 助手依然是“问答式”的：用户提问，模型响应。即便有了 Agent 和工具调用，执行时机仍然全由人类触发。在运维、开发、个人效率等场景中，真正的生产力跨越发生在 **AI 不用等你开口，而是帮你监控、判断、执行**——这就是 proactive 能力。

OpenClaw 社区的工具链（Agent 框架、MCP 插件体系、任务调度器）已经能支撑这种设计。本文以 GitHub 仓库维护为切入点，构建一个 proactive assistant：它会在你睡觉时自动给新 Issue 打标签、筛选可修复项并直接提交 PR。

# 2. 问题定义

我们想要一个主动式的 GitHub 助理，要求：

- 每隔 30 分钟检查指定仓库的新 Issue。
- 根据内容自动打上合适的标签（bug/enhancement/documentation）。
- 如果识别出明确的文档错误或错别字类问题，直接调用 API 修改对应的 Markdown 文件并生成 pull request。
- 所有写操作必须可审计、可回滚，不触发额外的自动化风暴。

# 3. 实现步骤

## 3.1 环境与工具

- OpenClaw Agent 框架（内置 scheduler 与 MCP client）。
- GitHub MCP server（提供 `list_issues`、`add_labels`、`get_file_contents`、`create_or_update_file`、`create_pull_request` 等工具）。
- 一个 GitHub Fine-grained personal access token，仅作用于目标仓库，授予 Issues、Contents、Pull requests 权限。

## 3.2 定时触发

在 OpenClaw 中通过内置 `cron` trigger 声明每 30 分钟的检查任务：

```yaml
triggers:
  - type: cron
    expression: "*/30 * * * *"
    action: proactive-github-run
```

`proactive-github-run` 是一段 Agent 任务描述，由 OpenClaw 解释执行，核心逻辑如下：

1. 调用 `list_issues(state: "open", sort: "created", per_page: 10, page: 1)` 获取最近 10 条 Issue。
2. 对每条未处理过的 Issue（通过 metadata 或 label `bot-reviewed` 判断），提取标题和正文。
3. 使用 LLM 判断 category（bug/doc/feature/other）和是否包含快速修复机会。
4. 如果 category 可归类，调用 `add_labels`。
5. 如果 LLM 判定为“documentation quick fix”，则调用 `get_file_contents` 读取对应文件，用 LLM 生成补丁，通过 `create_or_update_file` 提交到新分支，最后 `create_pull_request`。

## 3.3 安全边界的工程化加固

- **去重与幂等**：Issue 打标签后立即附加 `bot-reviewed` 标签作为状态锁；处理前检查该标签是否存在，避免重复操作。
- **写操作准入**：仅在 Issue 创建者不是自己（bot 账户）且 issue 没有 `wontfix` 等保护标签时，才进入自动修复分支。
- **PR 描述自动标注**：在自动 PR 的 body 中写入 `Automatic PR detected`，并嵌入回滚链接和触发 Issue 的 URL，方便人工关闭。
- **限流与分页**：MCP server 已处理 GitHub API 限流，但我们仍设置每批处理最多 3 次写调用（如打标签、创建分支），超出则下个 cron 周期再处理。

# 4. 踩坑点

1. **API 限流与 401 错误**  
   Fine-grained token 如果 scope 过窄会导致部分 API 403，排查时可通过 `get_rate_limit` 工具确认权限和余量。建议先使用经典 token 进行集成测试，再替换为细粒度 token。

2. **LLM 分类幻觉**  
   让 LLM 同时输出类别和置信度，低于阈值（如 0.7）则不自动打标签，只记录日志。实际发现，中英文混合的 Issue 容易触发幻觉，可以加一条 rule：“如果不确定，归类为 `triage-needed`”。

3. **文件修改路径推理错误**  
   从 Issue 描述提取文件路径再调用 `get_file_contents` 时，路径可能不完整。增加一层校验：如果返回 404，则调用 `search_code` 模糊搜索，再请求人工确认——这里我们选择直接退出自动修复并记录，防止错误修改。

4. **调度重叠导致并发写**  
   如果一次检查执行时间超过 30 分钟，下一个 cron 会再次启动，可能同时操作同一 Issue。为任务加上分布式锁（如 Redis 锁或简单文件锁），确保同一仓库同一时间只有一个 proactive run 在执行。

5. **通知噪音**  
   初期设置每次打标签都发 slack/邮件，结果瞬间刷屏。改为仅在产生 PR 或遇到异常时发送摘要，日常标签静默运行。

# 5. 可复用建议

把 proactive pattern 抽象为四步流水线，适用于任何服务：

- **Trigger**：cron、webhook、消息队列事件。
- **Query**：通过 MCP 工具拉取增量数据（GitHub issues、监控告警、日历事件）。
- **Decide & Act**：LLM 裁决 + 工具调用，附带安全控制（置信度、权限、幂等锁）。
- **Log & Notify**：所有写操作写入审计日志，异常推送通知。

你可以把这个模式复制到：

- 服务器异常自动创建工单并预填处理建议。
- 监控依赖库版本更新，自动跑 CI 并提 PR 更新 requirements。
- 日历空档检测并自动预约专注时间。

# 6. 总结

Proactive 不是让 AI 替你做决定，而是把重复感知、简单补救、非关键操作安静地完成，让你聚焦决策。本次在 GitHub 维护上的实践，验证了 “cron+MCP+LLM+安全护栏” 的组合是稳定可用的。注意权限最小化、操作幂等、人工回滚入口，就能让助手真正在你开口前把事办好，而不是闯祸。

---

