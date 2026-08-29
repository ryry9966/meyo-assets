---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 35267
source: 综合讨论
publishedAt: 2026-08-30
---

## 背景

OpenClaw 在处理自动化任务时，默认依赖每次会话的上下文。任务一结束，很多关键信息就会丢失：偏好的工具链、环境中不能碰的路径、输出格式约定、已经踩过的坑。下次再跑类似任务，又得重新交代一遍。

把这类信息写进 system prompt 能解决一部分问题，但很快会膨胀，而且 agent 无法在运行过程中沉淀新的经验。IDENTITY.md 的思路，就是给 OpenClaw 一个持久的、可被读取、也能在受控条件下回写的身份文件。它不是记忆库，而是一个可进化的接口。

## 问题

如果直接把 IDENTITY.md 当成“让 AI 随便记”，通常会把事情搞糟：

- **上下文膨胀**：无限制追加内容，最终超过模型有效窗口。
- **写入污染**：agent 把一次性结论当成长期规则写进去。
- **文件分裂**：全局和项目级各写一份，实际加载哪份不稳定。
- **并发冲突**：多个 agent 同时写，后写覆盖先写。

因此，更可靠的思路是：给身份文件加结构，并限制 agent 只能写哪些部分。

## 做法与步骤

### 1. 建立最小结构

我建议把 IDENTITY.md 控制在固定骨架内。一个可用的最小结构如下：

```markdown
---
role: automation_agent
last_updated: 2026-04-12
---

# Role
负责在仓库中执行自动化任务，不做未经确认的破坏性操作。

# Constraints
- 不修改 build/ 目录。
- 不直接推送到 main 分支。
- 所有命令使用非交互模式。

# Workspace
默认运行在 `~/projects/repo`。

# Known Issues
- ...

# Evolution Log
- 2026-04-12: 遇到依赖版本冲突，使用 lock 文件解决。
```

核心原则是：**Role、Constraints、Workspace 稳定不变；Known Issues 和 Evolution Log 允许演进。**

### 2. 在启动模板中挂载

在 OpenClaw 的配置文件中显式指定身份文件，而不是四处散布。示例：

```yaml
identity:
  file: .openclaw/IDENTITY.md
  auto_update: true
  allowed_sections: ["evolution_log", "known_issues"]
  max_entries_per_run: 2
  entry_max_chars: 500
  lock_file: .openclaw/.identity.lock
```

启动时只加载 Role、Constraints 和最近若干条 Evolution Log，完整文件仅在需要检索时读取。这能避免上下文被无意义的历史信息占满。

### 3. 限制写入权限

不要让 agent 直接编辑整个文件。只允许它追加到白名单区域，并规定格式：

- 每条经验必须包含日期、触发场景、解决方式。
- 单次运行最多追加 2 条。
- 不允许改动 Role、Constraints、Workspace。
- 写入前先做格式校验，失败则不落盘。

这样 agent 能沉淀经验，又不会把身份文件写乱。

### 4. 版本化与校验

把 IDENTITY.md 纳入 Git 管理，每次自动写入生成一个 commit。配合简单校验脚本检查 frontmatter、必需 section 和时间戳格式。这样即使某次写入有问题，也能快速回滚。

## 踩坑点

- **整文件塞进提示词**：身份文件一变大，启动上下文就超限。只加载关键段和最近条目。
- **偶发问题被写成规律**：某次网络失败可能不是稳定现象。建议先开 dry-run，让 agent 只输出拟写入内容，人工确认后再放开自动写入。
- **路径不一致**：固定唯一身份文件，优先使用 `.openclaw/IDENTITY.md`，并在配置中显式声明，不要同时维护全局和项目级两个版本。
- **并发写覆盖**：使用 append-only + lock 文件，或通过 Git 提交作为串行化层。
- **格式漂移**：没有校验脚本时，几次自动写入后格式就会变得不可解析。

## 可复用建议

- 身份文件只放**稳定偏好和已验证事实**，临时任务细节放独立日志。
- 把写入逻辑封装成函数，白名单化 section，不让 agent 任意编辑。
- 每隔一段时间做一次压缩：合并重复条目，删除过时 workaround，目标控制在 200 行以内。
- 反复出现的经验应该提炼成脚本、工具或 OpenClaw 插件，而不是无限堆进 IDENTITY.md。
- 用 Git 审计每次身份变更，能清楚看到 agent 从历史中学到了什么。

## 总结

IDENTITY.md 让 OpenClaw 在多次运行之间保持一致性，避免每轮都从零开始。但它不是万能记忆，也不该是无限膨胀的知识库。把它当成一个受控、可版本化、可回滚的身份接口，比让 AI“想记什么就记什么”更可靠。关键在于：结构稳定、写入受限、定期压缩、版本可查。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/30b648a03171ec62.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/c74c146ce4e09ccd.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/d16406263d248ec8.png)

