---
title: AI Agent 权限边界：给工具分级，而不是问“危不危险”
feedId: 34286
source: 综合讨论
publishedAt: 2026-08-23
---

## 背景

接入 MCP 之后，Agent 的工具列表很容易从 5 个涨到 30 个。文件、Git、数据库、HTTP 请求、消息推送，每一个都能自动执行时，问题就从“能不能调”变成了“该不该调”。

最常见的两个极端是：所有工具都 ask，人变成审批开关；所有工具都 allow，Agent 一次误判就删错文件或发错消息。权限边界不是某个工具的固有属性，而是工具、参数、运行环境、可逆性共同决定的策略。

## 问题：二元判断不够用

只看“危不危险”很难落地。`rm` 删 `/tmp/agent/` 下的临时文件风险很低，删 `/home/` 下的项目目录风险很高；`git push` 到 feature 分支通常可以接受，push 到 main 或 force push 需要确认。只按工具名做权限，要么太松，要么太紧。

工程上更实用的做法，是把工具按影响半径和可回滚性分级，再把分级映射到执行策略。

## 做法：四级分权 + 参数级策略

### 1. 先把工具分成四级

| 级别 | 类型 | 例子 |
|------|------|------|
| L0 | 只读、纯计算 | 读文件、查 API、跑本地脚本 |
| L1 | 可逆写、局部写 | 创建分支、写草稿、写临时文件 |
| L2 | 不可逆写或对外副作用 | 发送消息、合并 PR、删除文件、推送 main |
| L3 | 资金、权限、生产发布 | 付费、改 IAM、生产部署 |

### 2. 映射到执行策略

- L0：默认 allow，但保留审计日志；
- L1：allow + 记录摘要，必要时限制路径或数量；
- L2：默认 ask，或先 dry-run 再确认；
- L3：必须人工确认，且通常要二次校验。

在 OpenClaw 这类环境里，我通常不直接改 MCP server 的权限，而是在客户端侧加一层 policy。MCP server 负责暴露能力，策略层负责决定“这次能不能直接做”。类似下面这样，字段以实际版本为准：

```yaml
policies:
  - id: fs-read
    tools: [read_file, list_dir]
    policy: allow
  - id: fs-write-tmp
    tools: [write_file, delete_file]
    match:
      path_prefix: /tmp/agent/
    policy: allow
  - id: fs-write-home
    tools: [write_file, delete_file]
    match:
      path_prefix: /home/
    policy: ask
  - id: git-push-main
    tools: [git_push]
    match:
      branch: main
    policy: ask
  - id: external-http
    tools: [http_request]
    policy: ask
  - id: payment
    tools: [charge]
    policy: deny
```

### 3. 高风险任务先批计划，再执行

如果一个任务需要 20 步，每一步都弹确认，人会很快疲劳，最后直接全点允许。更稳妥的方式是让 Agent 先给出完整执行计划，计划内的工具自动执行，计划外新增工具再触发 ask。这比逐步确认少很多噪音，也不会把高风险动作藏在 20 个普通步骤里。

### 4. 加护栏，而不是只靠确认

确认框只是最后一道闸。更基础的是超时、最大步数、金额上限、路径白名单、API 速率限制和日志摘要。每个工具调用至少记录工具名、参数摘要、执行结果摘要，出问题能回溯。

## 踩坑点

1. **把“只读”当完全安全**。全表扫描会打爆数据库，读 `.env` 后写进日志外发也等于泄露。L0 也要限制范围和脱敏。

2. **只按工具名分级，不看参数**。同一个 `delete_file`，在 `/tmp` 和 `/home` 风险完全不同。参数级匹配必须做。

3. **审批太碎**。逐步确认会产生审批疲劳，用户最后闭眼点“允许”。用计划级审批替代步骤级审批。

4. **dry-run 不等于真实执行**。比如 `git push --dry-run` 不会触发服务端 hooks，真实推送仍可能失败或触发额外动作。dry-run 适用于预览，不替代关键操作的人工确认。

5. **只在 system prompt 里写“不要删重要文件”**。模型可能绕过或误解自然语言约束。权限必须在工具层 deny，而不是靠提示词。

## 可复用建议

- **新工具默认 ask 或 deny**，跑一周观察调用分布，再逐步放开。不要一上来就 allow。
- **用影响半径和可回滚性评分**，而不是“危不危险”这种二元标签。
- **策略配置版本化**。权限策略改了之后要有记录，和代码一样可回滚。
- **关键操作配回滚预案**。删除前先备份，合并前先创建回滚分支，发消息前先发草稿。
- **定期复盘误拒和误放**。哪些确认是多余的，哪些自动执行后来看是错的，用日志调整策略。
- **审计日志比确认框更重要**。确认只能挡当时，日志才能查之后。

## 总结

Agent 权限边界不是开或关，而是一套分层策略。自动化的目的不是完全无人，而是把人的决策成本降到最低。给工具分级、做参数级策略、先批计划再执行、加护栏和审计，这套组合比“全部 ask”或“全部 allow”更可持续。真正好用的 Agent，不是问得最多，也不是干得最多，而是每次干预都恰到好处。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/1a5174d56a198466.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/f2276e6dde6d07ea.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/b6aab66907ac6e3a.png)

