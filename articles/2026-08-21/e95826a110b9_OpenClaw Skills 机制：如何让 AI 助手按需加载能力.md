---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 33959
source: 综合讨论
publishedAt: 2026-08-21
---

## 背景

当 OpenClaw 从单任务脚本变成常驻助手，接入的工具、MCP 服务和自动化流程会越来越多：网页抓取、Git 操作、消息推送、数据库查询、浏览器自动化……如果把这些能力全部常驻到系统提示或工具列表里，每轮对话都要背着全部描述走。

工程中常见三类问题：

- 上下文膨胀：几十个工具描述吃掉大量 token，真正干活的内容反而被稀释。
- 误调用：工具描述互相干扰，模型容易选错能力。
- 维护失控：每次新增能力都要改主配置，系统提示越写越长。

Skills 机制要解决的不是“能不能调工具”，而是“什么时候才该把工具装进上下文”。

## 问题拆解

传统做法是把每个能力都注册为全局 tool，或者把 MCP 服务器全部常驻。这样看起来省事，但执行质量会随工具数量增加而下降。

OpenClaw 的 Skills 更像按需加载的卡片：平时只保留一个很小的索引，模型根据用户意图选中某个技能后，才把该技能的说明、脚本接口、约束和副作用注入当前会话。它和 MCP 不冲突——MCP 提供底层工具能力，Skills 负责把这些工具和本地脚本编排成可复用的执行单元。

## 做法 / 步骤

### 1. 建立 skills 目录

```text
skills/
  fetch-hn/
    SKILL.md
    run.sh
    README.md
  git-status/
    SKILL.md
    run.sh
```

每个技能一个目录，入口统一为 `SKILL.md`。

### 2. 定义 SKILL.md

```yaml
---
name: fetch-hn
description: Fetch top stories from Hacker News.
when_to_use: User wants headlines from Hacker News or HN top stories.
entrypoint: run.sh
allowed_tools: [curl]
timeout_sec: 30
version: 1.0.0
---
# Fetch HN

Return top 10 stories as a markdown list.
Do not use for generic tech news search.
```

描述里明确“什么时候用”和“什么时候不要用”，这比功能说明更重要。

### 3. 主 agent 只读 index

系统提示里不要放所有技能全文，只放索引：

```text
Available skills:
- fetch-hn: Fetch top stories from Hacker News. Use when user asks for HN headlines.
- git-status: Show current repo status. Use when user asks about uncommitted changes.
```

当用户说“看看 HN 今天什么新闻”，模型先根据索引选中 `fetch-hn`，再加载 `SKILL.md` 全文，然后执行 `run.sh` 返回结果。

### 4. 执行与回收

技能执行完就释放上下文，不常驻。对于频繁使用的技能，可以做短期缓存，但不要让缓存退化成全局工具。

## 踩坑点

- **when_to_use 写得太宽**：比如写成“Useful for getting information from the internet”，会导致每个问题都触发该技能。要在描述里加入明确边界和反例。
- **把技能做成薄封装 MCP 工具**：如果技能只是代理一下 MCP 调用，模型绕一圈又调回 MCP，没有降低复杂度。技能应该承载完整操作流程，比如“抓取 HN → 过滤标题 → 输出 markdown”，而不是只封装一个 HTTP 请求。
- **路径和工作目录混乱**：`SKILL.md` 里写 `python fetch.py`，但执行目录不在技能目录，导致找不到文件。建议入口脚本统一用相对技能目录解析路径，或在 `SKILL.md` 里显式声明 `working_dir`。
- **无条件信任脚本输出**：脚本返回什么就直接给用户，容易把错误堆栈或未处理异常带出去。应规定输出 schema，并检查退出码。
- **版本不固定**：技能改了行为却不更新版本，出问题后难回滚。给每个技能加 `version` 字段。

## 可复用建议

- 一个技能只解决一个可验证的任务，不要做“万能工具包”。
- 技能描述用三段式：做什么 + 什么时候用 + 绝不什么时候用。
- 给每个技能加 `--dry-run` 或 `--help`，让 agent 可以先验证再执行。
- 控制索引体积：单个技能描述不超过 100 词，索引总长纳入 token 预算。
- 记录 skill 加载耗时和 token 增量，方便事后审计。
- 危险操作如 push、delete、付款，技能入口必须显式要求确认。

## 总结

OpenClaw Skills 本质是一层上下文调度机制：用轻量索引保持全局精简，用技能卡片把指令、脚本和边界延迟到真正需要时加载。

落地后最明显的收益不是“工具更多”，而是误调用下降、上下文更干净、维护更清晰。它也不是银弹——如果技能描述随意写、脚本边界不清，Skills 只会退化成另一种形式的常驻噪音。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/3a6177ac69f63163.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/9f9a1227da808e9c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/4283d20b93ab6634.png)

