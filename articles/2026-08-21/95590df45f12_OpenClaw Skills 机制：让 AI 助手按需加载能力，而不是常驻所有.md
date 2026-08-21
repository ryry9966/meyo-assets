---
title: OpenClaw Skills 机制：让 AI 助手按需加载能力，而不是常驻所有工具
feedId: 33988
source: 综合讨论
publishedAt: 2026-08-21
---

## 背景

在 OpenClaw 里接 MCP、插件和自动化脚本的时候，很容易走到一个方向：把所有能力都挂进默认上下文。GitHub MCP、文件系统、浏览器控制、数据库查询、通知推送……一开始确实省事，助手上线就能干很多活。但跑了几天后，问题会集中暴露：

- 上下文被大量 tool schema 和系统提示占满，真正留给任务的窗口变小；
- 工具多了之后，模型选错工具的概率上升，尤其是功能相近的工具；
- 所有 MCP 服务常驻，启动变慢，资源占用和鉴权面都变大；
- 一旦某个插件或 MCP 出问题，整个 Agent 的稳定性都会被拖累。

OpenClaw 的 Skills 机制要解决的核心问题就是：**能力可以声明，但不要默认全部加载；真正要用的时候再注入。**

## 问题

全量加载的代价不是“多一点 token”，而是工程上的复杂度积累。

假设你的 OpenClaw 实例同时接入了 GitHub、Notion、Slack、文件系统和浏览器自动化。每一次对话，模型都要在这些工具里做选择。即使你只是让它改一个本地文件，GitHub 和 Slack 的工具描述也会参与路由。结果就是：

- 路由准确率下降；
- prompt 膨胀，成本上升；
- 每个 MCP 服务都要保持活跃连接；
- 权限面长期开放，安全风险增加。

更麻烦的是，某些能力之间有冲突或依赖。比如“发布 release”依赖 GitHub MCP 和本地 changelog 脚本，但如果两者都常驻，模型可能会绕过脚本直接调 GitHub API，导致流程不一致。

Skills 机制的思路是把这些能力拆成可声明、可匹配、可加载的单元，让助手先判断意图，再按需挂载。

## 做法 / 步骤

OpenClaw 的 Skills 通常是一个目录，包含技能描述、依赖和入口。一个典型结构如下：

```
skills/
  github-issue-triage/
    SKILL.md
    scripts/
      triage.py
    mcp.json
  local-file-ops/
    SKILL.md
    scripts/
      safe_replace.py
```

`SKILL.md` 是核心，用 frontmatter 声明技能的触发条件、依赖和权限。例如：

```yaml
name: github-issue-triage
description: 处理 GitHub issue 的标签、指派、关闭、评论等操作
when_to_use:
  - 用户提到 issue、PR、标签、指派、关闭
  - 需要读取或修改 GitHub issue 状态
not_when:
  - 只是查看代码或运行 CI
requires:
  mcp:
    - github-mcp
  scripts:
    - triage.py
permissions:
  github:
    - issues:read
    - issues:write
```

OpenClaw 启动时只扫描 `SKILL.md` 的 frontmatter，生成轻量索引。默认不加载具体工具，也不启动 MCP 服务。运行时流程大概是：

1. 用户输入后，先做技能路由，匹配 `when_to_use`；
2. 命中后加载对应技能的 system prompt、tool schema 和脚本说明；
3. 启动或连接所需 MCP；
4. 执行任务；
5. 任务结束后按策略卸载技能，释放上下文和连接。

配置层一般在 `openclaw.yaml` 里指定技能目录和加载策略：

```yaml
skills:
  path: ./skills
  max_loaded: 3
  auto_unload: true
  fallback: none
```

`fallback: none` 表示没有匹配到技能时，不加载任何额外能力，只用基础工具。

## 踩坑点

实际用下来，有几个点容易出问题。

**第一，描述写得太宽泛。**  
如果 `when_to_use` 写成“处理开发任务”，那几乎每次对话都会命中，技能变成常驻。描述要具体到用户会怎么说、任务边界在哪里。例如“创建或修改 GitHub issue”就比“GitHub 相关操作”更精确。

**第二，依赖没有显式声明。**  
有的技能内部脚本需要某个 Python 包，或者需要某个环境变量，但 `SKILL.md` 里没写。运行时加载成功，执行到一半报错。依赖一定要写在 frontmatter 或独立 `requirements` 里，让加载阶段就能校验。

**第三，MCP 连接不随技能卸载。**  
如果你在技能里启动了某个 MCP 服务，技能卸载后连接可能还挂着。尤其是全局 MCP 配置和技能级 MCP 混用的时候，容易造成资源泄漏。建议技能级 MCP 使用独立 session，卸载时显式关闭。

**第四，误触发比漏触发更危险。**  
漏触发只是助手说“我没有这个能力”，用户还能纠正；误触发可能让助手拿着不该有的权限去执行。所以在 `when_to_use` 里要写清楚 `not_when`，比如“查看代码不触发 GitHub issue 技能”。

**第五，频繁加载卸载带来延迟。**  
如果任务边界切得很碎，每次都要重新加载技能和 MCP，延迟会很明显。可以设置 `max_loaded` 保留最近用过的技能，或者把强相关的操作合并成一个技能。

## 可复用建议

一个比较实用的做法是：**把技能路由当成一个小型测试集来维护。**

为每个技能准备 5-8 条用户语句，其中包含正例和反例，定期跑一遍路由评估。例如：

- “帮我把这个 issue 标记为 bug” → 应命中 `github-issue-triage`
- “帮我看下这个文件的内容” → 不应命中 `github-issue-triage`

这比单纯靠感觉调描述可靠得多。

另外，技能内权限遵循最小化原则。不要因为方便就开放整个 GitHub 写权限，只给当前操作需要的 scope。技能描述里也可以加一句“不要执行列表之外的操作”，减少模型自由发挥。

日志也很重要。记录每次技能加载的原因、耗时和结果，便于事后排查“为什么这次没触发”或“为什么加载了这个技能”。这套机制能稳定跑起来，日志是性价比最高的投入之一。

## 总结

OpenClaw Skills 不是“把插件改成技能”这么简单，它改变的是能力管理方式：从常驻全集变成按需路由。核心在于声明清晰的触发边界、显式管理依赖、按任务粒度控制权限，并用测试集保证路由质量。

工程上真正省下来的不是几万 token，而是稳定性、可控性和扩展性。当你的 Agent 能力越来越多，还能保持不发散，这才是 Skills 机制的价值。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/ebad56a28ec11c13.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/b9b9a78b550a82b0.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/9ce0ec5f8af6d745.png)

