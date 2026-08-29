---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 35235
source: 综合讨论
publishedAt: 2026-08-29
---

# OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份

## 背景

在 OpenClaw 的 Agent 实践里，一个很常见的情况是：我们给 Agent 写了一段 system prompt，过两天加一点“不要用 sudo”，再过一周又加“输出先给结论”。这些规则散落在 system prompt、插件配置、记忆文件、甚至某个 MCP server 的描述里。单个 Agent 还能勉强运行，但一旦进入多 Agent 协作、插件调用、长期自动化任务，身份就开始漂移。

我遇到过几次典型的故障：重启后 Agent 忘了之前约定好的工作路径；两个 Agent 协同时一个按 root 用户习惯操作，另一个按项目用户习惯操作；插件拿到的是旧的行为偏好，结果生成了不适合当前环境的配置。这些问题不是模型能力不够，而是身份信息没有集中、没有版本、没有边界。

IDENTITY.md 的思路很简单：给每个 Agent 一个仓库内可读、可版本化、可审计的身份文件，作为它行为和记忆的单一事实来源。

## 要解决什么问题

1. **身份漂移**：system prompt 越改越乱，最后没人说得清这个 Agent 应该是什么。
2. **长期记忆不延续**：用户偏好、路径约定、错误经验只存在当前会话里。
3. **插件/MCP 无法取用身份**：工具侧拿不到“你是谁、你的限制是什么”，只能靠 prompt 间接传。
4. **协作不一致**：多个 Agent 各自为政，缺少共同的边界协议。

## 做法与步骤

### 1. 建立文件

在 Agent 工作区根目录放 `IDENTITY.md`。如果 Agent 通过 MCP 访问文件系统，也可以放到一个独立配置仓库，通过 resource 暴露。

### 2. 内容分块

建议固定五个区块，避免什么都往里塞：

```markdown
# IDENTITY

## Core Identity
- 角色：项目运维 Agent
- 语气：简洁、给命令前先给影响范围
- 不扮演：客服、销售

## Operating Principles
- 优先使用已有脚本
- 修改前先备份
- 禁止直接操作生产数据库

## Working Preferences
- shell: bash
- 默认分支: main
- 时区: Asia/Shanghai
- 输出路径: /workspace/output

## Evolution Log
- 2025-01-10: 禁止使用 sudo rm -rf，原因：误删 /tmp
- 2025-01-12: 增加备份步骤，原因：配置回滚困难

## Non-goals
- 不做前端页面生成
- 不处理支付相关数据
```

### 3. 接入运行链路

在 OpenClaw 的 pre-prompt 或 system prompt 里注入：

```text
Read IDENTITY.md from workspace root. It is your persistent identity.
Follow Core Identity and Operating Principles first.
When a conflict appears, ask before changing behavior.
```

若使用 MCP，可以写一个只读的 `identity-server`，把 `IDENTITY.md` 解析成 `identity://core`、`identity://preferences`、`identity://log` 几个 resource。插件和工具在需要时拉取，不用复制全文。

### 4. 让身份可进化

进化不是每次任务都改。建议提供 `/identity update` 或 `identity.update` 工具，追加 Evolution Log，并要求人工确认或 diff。大概流程：

```
用户确认 → Agent 生成变更条目 → 更新 Evolution Log → 提交 git
```

关键点是：身份变更必须有原因、时间、影响范围，不能默默覆盖。

### 5. 版本化

把 `IDENTITY.md` 纳入 git。每次变更至少写清楚 commit message，例如 `identity: add backup before config change`。这样当 Agent 行为突变时，可以先看身份文件改动，而不是翻几页 system prompt。

## 踩坑点

**文件过长**  
我见过把几十条少用偏好全部塞进 IDENTITY.md 的做法，结果每次请求 token 暴涨，Agent 反而抓不住重点。控制在 200-400 行以内，核心身份尽量 20 行内。少用的规则放外部 SOP 文件，需要时再读取。

**把秘密写进去**  
不要在 IDENTITY.md 里写 token、密码、内网拓扑。身份文件可能被日志、调试输出、插件 provider 打印出去。密钥放 secret manager，通过 MCP 按需取用。

**没有变更记录**  
直接覆盖文件等于没有进化。出了问题时不知道哪次修改导致。Evolution Log 至少要能回答：什么时候、改了什么、为什么。

**身份与权限脱节**  
文件里写“可以操作生产”，但实际沙箱只给只读权限，Agent 会反复尝试失败。身份文件描述“能做什么”要和实际授给工具链的权限一致，否则会产生错误预期。

**过度进化**  
让 Agent 每次任务都追加“我应该更谨慎”会导致身份文件膨胀且不稳定。建议限制频率：例如只有在任务失败复盘、用户明确要求、或每周 review 时才允许变更身份。

## 可复用建议

- **先跑最小版**：只写 Core Identity 和 3 条 Operating Principles，跑一周再扩展。
- **固定标题**：`## Core Identity`、`## Operating Principles`、`## Working Preferences`、`## Evolution Log`、`## Non-goals`，便于脚本解析。
- **多 Agent 分层**：共享约束放 `SHARED_IDENTITY.md`，每个 Agent 自己的 `IDENTITY.md` 只写差异。
- **用 MCP 只读暴露**：避免 Agent 直接改源文件；提供 `identity.read` 和 `identity.propose_update`，更新走 PR 或确认流。
- **审计优于信任**：把 Evolution Log 当成操作记录，不只是一个 memo。

## 总结

IDENTITY.md 并不是另一种 system prompt，而是 Agent 的长期身份层。它解决的不是“这次回答好不好”，而是“这个 Agent 是不是可预期、可回滚、可协作”。在 OpenClaw 这类 Agent 工程里，身份稳定比单纯聪明更重要。建议从一个小文件开始，把“身份可进化”变成可审计的工程动作，而不是只停留在 prompt 里的一句话。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/77dc0d48bae605ea.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/1b09483f771a1799.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/1522c2188edd1c2b.png)

