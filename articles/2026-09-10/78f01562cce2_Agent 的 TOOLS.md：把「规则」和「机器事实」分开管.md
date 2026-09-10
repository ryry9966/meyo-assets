---
title: Agent 的 TOOLS.md：把「规则」和「机器事实」分开管
feedId: 36904
source: 综合讨论
publishedAt: 2026-09-10
---

## 背景

OpenClaw 的 agent 工作区里有一组 Markdown 会被注入模型上下文：AGENTS.md 管行为约定，SOUL / IDENTITY 管人格，而 TOOLS.md 的定位常被误解——它其实是给 agent 看的**本机环境说明书**。默认工作区模板甚至把它放进了 `.gitignore`，原因很直白：内容天然是 per-machine 的。

单机使用时感知不强。一旦你有两台以上机器（台式机 + 笔记本 + 一台 VPS），或者 macOS / Linux / WSL 混用，环境差异就开始收税了。

## 问题

常见的三种错误姿势：

1. **全写进 AGENTS.md**。共享文件里出现了 `/Users/xxx/...` 和 `brew`，另一台 Debian 机器上的 agent 照着执行，报错还得你来收拾。
2. **每次口头告诉 agent**。它不跨会话记忆，同一个坑每次重踩。
3. **写死在 skill / 脚本里**。换机器直接失效，agent 还不知道，继续用旧路径和旧参数。

本质问题是把「跨机器稳定的规则」和「随机器变化的事实」混在了一个层次。

## 做法

原则一句话：**AGENTS.md 写规则，TOOLS.md 写事实**。TOOLS.md 是写给模型读的，要短、要陈述句、不要散文。骨架示例：

```markdown
# TOOLS.md — 本机环境事实（勿提交 git）

## runtime
- node v22（nvm 管理）；系统 python 3.10，别用它跑脚本

## 包管理
- 本机 macOS 用 brew；VPS 是 Debian 系，用 apt

## 路径
- openclaw 配置: ~/.config/openclaw/
- 常用工作区: ~/work/*

## MCP / 插件
- browser MCP 本机走本地 Chromium；VPS 上禁用（无显示环境）

## 已知坑
- shell 是 zsh，rc 在 ~/.zshrc；VPS 上 docker 需要 sudo
```

落地步骤：

1. 每台机器初始化时，让 agent 自己盘点环境并生成 TOOLS.md 初稿，人工删改。
2. `.gitignore` 掉它，仓库里只留 `tools.md.example` 模板。
3. 换机或升级后重跑一次盘点，diff 再更新。

盘点 prompt 可以固化成 skill：

```
检查本机环境，把以下事实写入 TOOLS.md：各 runtime 版本与安装方式、
包管理器、关键路径、shell 类型、MCP/插件可用性、权限限制。
只写事实，不写建议。
```

## 踩坑点

- **别放密钥**。TOOLS.md 会进模型上下文，把它当 prompt 看待，不是 secrets 存储。
- **别超长**。建议控制在 100 行内，每多一行都在稀释注意力。
- **别和 AGENTS.md 冲突**。同一问题两处说法，agent 听哪边看你运气，source of truth 只能有一个。
- **会过期**。node 升级、路径迁移后它就是谎言来源，我的习惯是大版本升级后强制重跑盘点并 diff。

## 可复用建议

- 判断标准：这条信息换台机器还成立吗？成立 → AGENTS.md；不成立 → TOOLS.md。
- 各机器 workspace 独立维护，**不要**用符号链接"共享"TOOLS.md——那等于回到错误姿势一。
- 把盘点 prompt 存成固定 skill 片段，新机器冷启动直接调用。
- 团队场景：`tools.md.example` 进仓库，新人 clone 后照着填，agent 第一次运行就有正确的环境认知。

## 总结

TOOLS.md 的价值不在文件本身，而在它逼你把配置分成两层：稳定规则进共享文件，机器事实留在本地。做对这件事，agent 在哪台机器上都不会拿错路径、用错包管理器。花十分钟给每台机器写一份诚实的环境清单，比之后无数次纠正它便宜得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/e514ff2f78350d97.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/6acbe210fc97a72e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/ebf6fe9d0ae789a1.png)

