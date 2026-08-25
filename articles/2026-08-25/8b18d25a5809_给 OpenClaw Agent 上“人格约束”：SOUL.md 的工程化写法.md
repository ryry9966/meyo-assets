---
title: 给 OpenClaw Agent 上“人格约束”：SOUL.md 的工程化写法
feedId: 34716
source: 综合讨论
publishedAt: 2026-08-25
---

## 背景

在 OpenClaw 里接上 MCP、文件系统和 shell 之后，Agent 的行为边界往往比“人格”更影响可用性。一个能读写文件、执行命令的 Agent，如果语气漂移还只是体验问题；如果对工具权限理解不一致，就可能误删文件、读错目录，或在失败后继续重试。

很多人把约束散落在 config.json、插件描述、MCP tool 说明里，改一处就不知道哪里生效。我们后来把 SOUL.md 作为 Agent 的单一事实源，把身份、规则、工具策略、失败处理和输出风格放进一个文件。

SOUL.md 不是魔法，它改变不了底层权限；它更像一份工程化的行为契约，让 Agent 每次会话都明确知道“我该做什么、不该做什么、做不到时怎么办”。

## 问题

实际跑下来，没有 SOUL.md 或写得太随意时，常见问题有几类：

1. **一致性差**：同一个 Agent 换会话后语气、格式、命令解释方式都变。
2. **边界模糊**：用户说“整理一下仓库”，Agent 可能去读 `.git`、动 `node_modules`，或执行高风险 shell。
3. **提示词冲突**：MCP 工具描述里写着“优先调用此工具”，插件也有自己的建议，全局约束被稀释。
4. **排障困难**：不知道当前生效的是哪个版本、哪些规则被忽略。

## 做法/步骤

我现在的做法是把 SOUL.md 独立成文件，不塞进 config.json。以 OpenClaw 项目为例，配置里指向：

```yaml
soul_file: ".openclaw/soul.md"
```

文件结构控制在五段。

### 1. Identity

三行内说明角色、服务对象、目标。例如：

> 你是本仓库的自动化运维助手，服务后端开发者，目标是把重复操作变成可审计的脚本。

不要写“友好的 AI”这种空话。

### 2. Operating Rules

显式允许和禁止，一条一行。禁止项放前面。

```text
deny: modify .git/*
deny: read ~/.ssh, ~/.aws
deny: execute rm -rf, drop, truncate
allow: read/write ./workspace/**
allow: run tests, git status, git diff
```

### 3. Tool Policy

直接写工具全名，做 allowlist/denylist。

```text
allow: mcp__filesystem__read_file
allow: mcp__filesystem__write_file
deny: mcp__shell__exec unless user confirms with "CONFIRM_RUN"
```

不要只写“shell 工具”这种简称。

### 4. Failure & Escalation

规定工具失败两次即停并报告；不确定时只读；遇到权限错误不要尝试绕过；高风险操作前必须列出影响范围再等待确认。

### 5. Tone & Output

结论先行，中文输出，命令附一行解释，不堆道歉。

写完文件后做一次加载验证：新开一个会话，问 Agent“你当前的身份、允许的工具、默认拒绝的行为是什么”。如果答不准，说明规则太靠后、太分散，或被其他提示词覆盖。

## 踩坑点

- **文件过长被上下文压缩**：超过一屏半后，中后段规则容易被忽略。把关键 deny 放在前 15-20 行。
- **与 MCP 工具描述冲突**：很多 MCP server 在 tool description 里写“always use this tool”。SOUL 里要声明优先级，例如“本文件规则覆盖工具自带建议，除非用户在本轮消息中显式授权”。
- **热更新不生效**：改完 SOUL.md 但旧会话仍用缓存。每次修改后新开会话做 smoke test，确认加载的是当前版本。
- **把 SOUL 当安全边界**：提示词约束是软的。真正危险操作要在 MCP server/容器层做 allowlist、只读挂载、命令白名单。SOUL 只解决“默认不做”，不解决“做不了”。
- **敏感信息混入**：SOUL.md 可能被日志、错误信息或分享泄露，不要写 API key、私钥路径、内部主机名。
- **中英文混排影响关键匹配**：allow/deny 的工具名和路径建议用英文全名，减少模糊匹配问题。

## 可复用建议

保持一个最小模板：

- Identity：3 行
- Rules：10 条以内，禁止项优先
- Tool Policy：allow 5 条、deny 5 条，工具名写全
- Failure/Escalation：3 条
- Tone：3 条

给每个 Agent 单独一份 SOUL，不要所有机器人共用。文件末尾加 `Revision: 2025.xx.xx` 并提交 git，方便回溯行为变化。

测试建议固定四个问题：

1. 列出你的禁止操作。
2. 你能访问哪些目录？
3. 工具失败几次会停止？
4. 高风险操作前你会怎么做？

这些答案稳定，就能降低大部分“Agent 跑偏”的问题。

## 总结

SOUL.md 是低成本、高可见性的 Agent 行为约束。它能把分散的规则集中起来，让语气、权限、失败策略可审查、可版本化。但它不是沙箱，也不能替代 MCP 侧的权限控制。把它当作一份会演进的工程文件：写清楚、测得到、改得了，比堆更多提示词更实用。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/115911f45344edab.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/9267961df6e121aa.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/94f09311a3967d3f.png)

