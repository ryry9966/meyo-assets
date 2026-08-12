---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 32785
source: 综合讨论
publishedAt: 2026-08-12
---

## 背景：当一次性 Prompt 不再够用

在 OpenClaw 这类 Agent 框架里，我们习惯用 Markdown 目录下的 `system.md` 或 `personality.md` 定义 AI 的行为边界。问题是，这些文件通常是静态的。一旦任务链拉长、交互轮次增多，固定身份就开始成为瓶颈——AI 永远在用第一次见到的那套偏好做决策，无法从昨天的错误里长记性，也不会因为用户换了工作流而调整自己的角色。

手动改文件当然可行，但那等于把“自动化”这三个字吞回去。于是有了 `IDENTITY.md`——一个允许 Agent 在运行时重写自己身份的核心文件。

## 问题：一成不变的身份如何拖慢自动化

典型的场景是：你让 OpenClaw Agent 管理你的技术笔记仓库。第一天它很勤快，按“简洁优先”的风格整理 Markdown。一周后你发现新的笔记更适合“带详细标签 + 上下文综述”，于是你口头告诉它调整。Agent 下一次执行确实改了，但它没有持久化这个偏好。再往后，当它重启会话或遇到多任务分支时，因为 `system.md` 没变，行为又回到了最初的状态。你的每一次纠偏都只作用在那一个上下文窗口里。

这不是记忆力问题，而是**身份没有跟着使用场景进化**。对 MCP 工具链、定时任务（schedules）或插件驱动的长周期自动化来说，丢失这种演进能力，就意味着需要频繁的人工干预来维持行为一致性。

## 做法：用 IDENTITY.md 实现可进化的身份

OpenClaw 核心逻辑允许你在 Workspace 根目录放置 `IDENTITY.md`，并通过内置的“自省 + 文件操作”能力让 Agent 在特定条件下更新它。下面是一个经过实战沉淀的步骤。

### 1. 设计可更新的字段

不建议让 AI 能随意重写整个文件。把 `IDENTITY.md` 分成**不可变区**和**可变区**。不可变区写死核心目标与安全准则，可变区存放“当前工作风格”“最近学到的偏好”、“有效工具速查”等动态内容。

```markdown
# Core (DO NOT CHANGE)
- Goal: Maintain tech notes repo with high signal-to-noise ratio.
- Constraint: Never delete user content unless explicit backup confirmed.

# Evolvable
## Style preference
- Currently: Detailed with tags (updated 2025-03-17)
## Recent lessons
- Avoid auto-linking internal drafts; user prefers manual review.
## Tool preferences
- Use `ripgrep` over built-in grep for large repos.
```

### 2. 配置读取入口

在 OpenClaw 的 `system.md` 或启动 Prompt 中增加一条指令，要求 Agent 每次决策前先读取 `IDENTITY.md`，并严格遵守其中可变区的最新内容。这样可变区就成了“实时生效的行为引擎”。

### 3. 设置触发更新的条件

通过 OpenClaw 的 `schedule` 或任务完成后的钩子，让 Agent 根据交互日志生成一份“自省摘要”，然后调用文件系统工具（如 `write_file`）更新可变区。触发条件可以这样定义：

- 用户连续两次对同类任务给出风格纠正；
- 任务失败且根因分析指向过时的偏好；
- 手动命令 `@agent update identity`。

关键是把**更新幅度控制在最小必要粒度**。例如，只追加一条教训而不重写整个偏好段落，避免丢失历史上下文。

### 4. 版本控制与回滚

给 `IDENTITY.md` 加 Git 跟踪是最简单可靠的保护。每次更新后，Agent 自动生成 commit 信息如“Update style preference based on task XYZ feedback”。如果发现身份漂移（后文详述），直接 revert 到上一个稳定版本即可。

## 踩坑点：进化不代表失控

实际使用中，最容易碰到三个坑。

**身份漂移**  
Agent 过度追求“适应用户”，导致可变区逐渐偏离核心目标。例如，用户随意说了一句“今天随便点”，它就把风格永久改成极简，以后再也回不来。对策是在不可变区设定硬边界，并在更新 Prompt 中加入 constraint：“每次变更必须与不可变区的 Goal 保持一致，若判断可能偏离，仅记录为临时建议而不修改文件”。

**更新过频导致 I/O 抖动**  
短时间反复更新会造成文件写入竞争，尤其在多 Agent 协作时。设置最小更新间隔（如至少间隔 5 个任务）和防抖机制（累积 3 条同类型变化再写入）能有效缓解。

**安全与代理权限**  
让 Agent 有权限修改自己的“大脑”文件，本质上是一次信任升级。一定要通过 OpenClaw 的权限策略（如 `allowed_paths`）限制其只能操作指定文件，并且永远不要让它可以删除不可变区标记。保留人类审批接口作为最终防线。

## 可复用建议

如果你不想每次都从零设计，可以考虑将可变区抽象成一个模块，用 YAML front matter 或简单的键值对表示偏好，Agent 只更新这些键的值。这样更结构化，也能直接用脚本校验。

另一个思路是将可变区内容与长期记忆系统联动。比如把最近教训存入本地向量库，`IDENTITY.md` 只保留指向检索结果的摘要，减少文件体积，又能调用更丰富的上下文。

对于团队共用 Agent 的场景，可以将 `IDENTITY.md` 拆成“团队基础身份”与“个人覆盖层”，Agent 根据当前操作用户加载对应的覆盖，形成一级统一的团队规范 + 二级个性化行为，避免多人互相覆盖。

## 总结

`IDENTITY.md` 并不神奇，它本质是一个被 Agent 自己维护的、带版本控制的运行时配置文件。但它解决了开放域自动化里一个很实际的问题：Agent 不能每次都从白板开始。给 AI 一个可进化的身份，意味着它能像熟练的工程师一样，在反复协作中磨合出最佳工作默契，而不是永远停留在静态的 System Prompt 时代。在 OpenClaw 生态里，这个文件搭配 `git`、权限控制和合理的更新约束后，就成了低成本、高回报的“代理肌肉记忆”，值得在需要长周期自动化的项目里落地一试。

---

