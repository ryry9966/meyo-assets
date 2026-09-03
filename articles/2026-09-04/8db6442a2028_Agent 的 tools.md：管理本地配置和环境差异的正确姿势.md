---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 35978
source: 综合讨论
publishedAt: 2026-09-04
---

## 背景

跑 OpenClaw / Agent 时，除了 MCP 暴露的工具，机器上还有大量本地能力：ffmpeg、jq、自研 CLI、内部脚本……模型并不天然知道这些工具的存在和用法。工作区里的 tools.md 就是给 Agent 看的"本地工具手册"：每次会话它都会读，并据此决定调什么、怎么调。

## 问题

一旦涉及多机环境，tools.md 很快失真：

- 同一个工具，Mac 上在 `/opt/homebrew/bin/`，Linux 服务器在 `/usr/local/bin/`，容器里可能根本没装；
- 版本不一致导致参数行为不同，Agent 按文档调用却报错；
- 拿不到准确信息时 Agent 开始"试探"：反复 `which`、`--help`，烧 token 也烧时间；
- 更糟的是有人把 token、绝对路径直接写进共享的 tools.md，随仓库同步出去。

本质：tools.md 被当成了静态文档，而它描述的是一个会漂移的环境。

## 做法

1. **分层**：tools.md 只放"稳定事实"——工具用途、调用方式、前置条件、失败表现；机器相关内容放 `tools.local.md`（进 `.gitignore`）或用环境变量间接引用。
2. **写"可执行"而不是"描述性"**：每条工具给一行可直接复制运行的命令，别写散文。Agent 要的是 invocation，不是介绍。
3. **用自定位命令**：优先 `uvx` / `pipx` / `npx` / 包装脚本，让路径在运行时解析，而不是写死绝对路径。
4. **加 smoke test**：每个工具附一条验证命令（如 `ffmpeg -version | head -1`），Agent 调用前先跑，失败即回退，避免在坏环境上叠加错误。
5. **定期修剪**：文档里的每个工具都应该真实存在。过时条目比没有更危险——模型会优先信文档。

## 踩坑点

- **硬编码密钥**：tools.md 常被同步、备份，任何 credential 都不该出现，用 `$ENV_VAR` 占位；
- **写太长**：Agent 每次都读，300 行手册会稀释关键信息，控制在能一眼扫完的篇幅；
- **只写 happy path**：不写超时、权限要求、输出怎么解析，Agent 第一次失败就会自己乱编参数；
- **多机共用一份**：以为"统一文档"省事，实际是三台机器互相污染假设，出问题还难定位是哪台的环境。

## 可复用建议

- **把 tools.md 当代码**：改动手动 review，配一个 lint 脚本 grep 掉 `sk-` 前缀、绝对路径等敏感模式再提交；
- **模板化**：团队维护一份 `tools.template.md`，新机器只需要填 local 层，十分钟接入；
- **版本钉死**：文档里写清依赖版本，和 smoke test 一起构成最小验收标准；
- **全局与项目两级**：通用工具放全局工作区，项目专属工具放项目内，避免一份文件承担所有上下文。

## 总结

tools.md 的价值不在于"写了多少"，而在于 Agent 读完后的第一发命令命中率。把稳定知识与易变环境分层，把描述换成可执行、可验证的命令，再配上定期修剪，它就从一份容易腐烂的笔记，变成环境差异的单一事实来源。这比任何提示词技巧都更省钱、更稳。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/3ac046ef91eefeb4.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/da2dcd339ab87276.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/ce91a6532dd6f40f.png)

