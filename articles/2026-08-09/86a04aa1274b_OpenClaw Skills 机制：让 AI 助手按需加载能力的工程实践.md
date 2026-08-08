---
title: OpenClaw Skills 机制：让 AI 助手按需加载能力的工程实践
feedId: 32173
source: 综合讨论
publishedAt: 2026-08-09
---

## 背景：工具膨胀与上下文污染

在基于 LLM 的 Agent 开发中，「把所有能力都塞进系统提示和工具列表」是初期最容易走的捷径。但很快你就会撞到三堵墙：

1. **上下文长度上限** —— 即使模型支持 128K，长工具描述、示例、参数 schema 也会轻易吃掉几万 token。
2. **注意力衰减** —— 可用工具超过 20 个后，模型误选工具的概率明显上升，尤其在非英语场景下。
3. **调试负担** —— 修改一个能力需要重新梳理依赖，回归测试成本随工具数量指数级增长。

OpenClaw 作为面向自动化的 Agent 框架，提出了一套轻量级的 **Skills 按需加载机制**，让助手不必在每次对话中都携带着所有能力，而是像微服务那样，按请求上下文动态激活。我最近在内部项目中落地并跑了一段时间，下面是把这套实践整理出来，供社区参考。

## 问题：静态工具绑定为何失效

一般 Agent 的实现模式是：

```
System Prompt + 所有工具定义 + 用户消息 → LLM → Tool Call
```

这在小规模场景下完全可行，但当你的助手需同时处理「服务器监控」「CI 触发」「工单系统」「知识库检索」四类完全不同领域的能力时，一次会话里可能只需要其中 1~2 类。静态绑定导致大量无关工具描述悬浮在上下文中，不仅浪费 token，还会误导模型「幻想式调用」一个根本不相关的工具。

简言之，我们需要从「全量预装」变为**上下文感知的懒加载**。

## 做法：OpenClaw Skills 机制拆解

OpenClaw 的 Skills 设计核心是三个概念：

- **Skill Manifest**：每个能力的声明文件（YAML/JSON），定义触发条件、依赖、工具列表、示例对话片段。
- **Skill Registry**：运行时的能力注册中心，可本地文件、Git 仓库或远程服务加载。
- **Orchestrator**：负责接收用户输入，在会话上下文中匹配应激活的 Skills，并在本轮推理前组装最终的工具列表。

### 关键步骤

#### 1. 编写 Skill Manifest

每个 Skill 一个独立目录，最少包含 `manifest.yaml` 和可选的 `system_prompt.md`、`tools.json`。一个典型的 manifest 结构如下：

```yaml
name: github-issue
version: "1.2.0"
description: "Manage GitHub issues: create, list, close"
triggers:
  keywords: ["issue", "bug", "task", "assign"]
  intent: "github-issue-management"
  templates:
    - "create an issue in {repo}"
    - "list open issues assigned to me"
dependencies:
  skills: []  # 可依赖其他 Skill
  env_vars: ["GITHUB_TOKEN"]
tools:
  - name: create_issue
    description: "Create a new GitHub issue..."
    parameters: {...}
```

`triggers` 是激活的核心：关键词、意图标签、模版句子都会参与匹配。实际匹配依靠嵌入相似度 + 规则混合，而非纯 LLM 判断，这样能降低延迟与误判。

#### 2. 注册到 Registry 并配置策略

Skills 可以放在本地 `skills/` 目录下，OpenClaw 启动时自动扫描并注册。你也可以通过 Git 子模块引入社区 Skill。加载策略可以配置：

- **session-scoped**：某 Skill 一旦激活，整个会话期间保持可用（适合连续操作场景）。
- **turn-scoped**：每轮根据用户最后一条消息重新决定激活集合（更省 token）。
- **hybrid**：将高频 Skill 设为常驻，其他按需。

我的工程选择是 hybrid：把工具检索、环境查询这类基础能力设为常驻，业务专用能力（如工单、发布）按 turn-scoped 激活。

#### 3. 在对话流程中利用 Orchestrator

每次用户请求到达，Orchestrator 会先跑一个轻量匹配：从 Registry 拉取所有 Skill 的 trigger 特征，与当前输入做比较，得到一个激活候选集。然后根据策略决定本轮最终激活的 Skills，将它们的 `system_prompt` 和 `tools` 合并到发送给 LLM 的请求中。

这里有个性能细节：匹配过程不是每次都加载完整 manifest，而是预先构建的索引（trie + embedding 缓存），匹配耗时通常在 20ms 以内。

## 踩坑记录与真实教训

实际落地中遇到的问题远比文档里写的多。

**1. “漏激活”最致命，而不是“多激活”**

早期我们倾向少激活以减少 token，结果经常漏掉用户隐性意图。比如用户说「上次那个 PR 合并后，环境好像有问题」，明面上是环境问题，但背后可能依赖 github-issue 和 deploy-log 两个 Skill。单靠关键词匹配会漏掉 deploy-log。后来我们引入二次确认机制：如果本轮调用返回的 tool result 中包含未激活 Skill 的明确信号（如错误信息里出现「deployment failed」），Orchestrator 会在下一轮自动补激活相关 Skill。这个「补偿性激活」极大降低了漏报。

**2. Skill 间的 tool 命名冲突**

多个 Skill 可能定义了同名的 tool，比如 `list_files`。如果同时激活，LLM 的 tool_choice 会混乱。解决办法是在 manifest 层使用命名空间：tool 名暴露给 LLM 时自动加上 Skill 前缀，如 `github_issue_list`。这只是约定，严格要通过 Registry 的冲突检测来拒绝同前缀的 Skill 同时入驻。

**3. 上下文窗口的二次膨胀**

动态加载看似省 token，但如果某次会话触发了大量 Skill，累积的 system_prompt 和 tool 描述仍会打满窗口。实践中必须设定一个 **max_active_skills** 阈值（我设的是 5）。当超出阈值时，按优先级（用户显式触发 > 隐式匹配分数）截断，并告知 LLM 本轮的可用能力范围，避免它假设自己有全量能力。

## 可复用建议

1. **Skill 粒度做到原子化**：一个 Skill 只解决一个领域问题，避免「万能 Skill」。这既提升了匹配准确性，也方便独立测试和版本管理。
2. **提供触发示例而非纯关键词**：在 manifest 的 `templates` 中给出 5-10 条真实用户可能说的句子，比放一堆关键词更能提升匹配召回。
3. **记录激活日志并做离线分析**：把每次匹配结果、激活集、用户反馈（任务完成度）落到日志，离线跑一下 embedding 相似度分布，能找到很多优化 trigger 的数据依据。
4. **为每类 Skill 定义生命周期**：部分 Skill 需要会话级状态（如数据库连接），部分是无状态的。利用 `dependencies.env_vars` 清晰声明，避免运行时抛缺少环境变量的奇怪错误。

## 总结

OpenClaw Skills 机制本质是把 Agent 的工具集从「单体应用」变成了「可插拔模块」。它并不能解决 LLM 推理本身的问题，但通过按需加载，让助手的上下文更干净、模型选择工具更精准、系统扩展更线性。如果你目前的 Agent 工具数量已经超过 15 个，或者需要多人协作开发独立能力，那么值得认真引入这套机制，而不是继续在系统提示里堆砌 YAML。

---

