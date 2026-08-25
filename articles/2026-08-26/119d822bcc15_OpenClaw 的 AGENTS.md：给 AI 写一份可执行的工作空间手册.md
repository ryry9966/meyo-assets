---
title: OpenClaw 的 AGENTS.md：给 AI 写一份可执行的工作空间手册
feedId: 34785
source: 综合讨论
publishedAt: 2026-08-26
---

## 背景

OpenClaw 接入几个 MCP 和一个插件之后，工作空间就不再是单纯的代码目录：有缓存、原始数据、输出产物、脚本、配置。每次新会话，模型都要重新理解目录结构，或者直接按默认习惯乱翻。AGENTS.md 是放在工作空间根目录的说明文件，它真正的价值不是“文档”，而是让 agent 在动手前先读一份稳定约束。

## 问题

没有 AGENTS.md 时，常见故障非常集中：

- agent 全局搜索或递归读取 node_modules、.git、dist，token 迅速耗尽。
- 把临时文件、导出结果写进源码目录，或者反过来覆盖了只读数据。
- 明明通过 MCP 能安全操作文件，却退回 bash 里 cat/find，破坏权限模型。
- 每次会话都要重复“不要碰 data/raw、不要自动 commit”，成本高且不稳定。

这些问题根因不是模型能力不足，而是工作空间缺乏可被稳定读取的接口约束。

## 做法/步骤

1. 在项目根目录创建 AGENTS.md。OpenClaw 里通过系统提示或插件入口增加一句：进入工作目录后先读 AGENTS.md，并在回复中复述 hard rules。这样可以把读取变成固定动作，而不是偶尔。
2. 文件结构保持短。建议分四块：
   - Workspace Map：目录职责
   - Hard Rules：硬约束
   - Tool & MCP：工具使用约定
   - Task Protocol：执行顺序
3. 给出一个精简版示例如下。

```markdown
# AGENTS.md

## Workspace Map
- src/       # 主代码，可写
- tests/     # 测试，可写
- data/raw/  # 原始数据，只读
- data/out/  # 处理结果，可写
- scripts/   # 自动化脚本，可写
- .cache/    # 禁止读取和写入

## Hard Rules
- never read node_modules/, .git/, .cache/, dist/
- never modify data/raw/
- before writing any file, check git status
- for files > 2MB, use head or wc before full read
- do not run git commit unless explicitly requested

## Tool & MCP
- use filesystem MCP for file ops; do not use bash cat for source files
- use search MCP only with path filter; never global search without filter
- if a MCP call fails twice, stop and ask before fallback

## Task Protocol
1. read target files first
2. list changes before editing
3. after editing, show diff summary
4. do not create new top-level directories without asking
```

4. 写完做一次验证：让 agent 执行一个简单小任务，看它是否先读 AGENTS.md、是否避开了 node_modules。验证失败就收紧措辞，或把关键规则提到系统提示。

## 踩坑点

- **太长**：超过 100 行后，模型经常只读开头，后面的规则失效。保持 60-100 行以内，用祈使句。
- **用词太软**：should、try to、prefer 对 agent 的约束力很弱。能明确就写 must、never、always。
- **绝对路径**：换机器、容器路径变化后规则全废。用相对路径。
- **与系统提示冲突**：如果系统提示层有相互矛盾的默认行为，AGENTS.md 经常被覆盖。关键硬规则需要同步到系统提示或插件配置，而不是只放在文件里。
- **多角色混淆**：同一个工作空间里不同 agent 权限不同，不要写一个万能文件。可以拆成 AGENTS.dev.md、AGENTS.ops.md，或按 section 标记适用角色。

## 可复用建议

- 把 AGENTS.md 纳入 git 版本管理，改完要像代码一样 review。
- 每新增一个 MCP 或插件，就同步更新 Tool & MCP 段；否则 agent 会继续沿用旧工具。
- 在启动指令里固定写：read AGENTS.md first, then reply with a one-line plan。这比只写“请遵守 AGENTS.md”有效。
- 定期做规则回归：让 agent 读文件后用一句话复述 hard rules，能很快发现哪些规则被忽略。
- 目录变大后，不要把所有说明塞进 AGENTS.md，只保留硬规则和索引，细节放到 docs/。

## 总结

AGENTS.md 是给工作空间加的一层接口约束，不是给模型看的散文。先解决“别乱翻、别乱写、用对工具”这三件事，再谈复杂自动化。一个稳定、可预期的 agent，比一个每次都要重新教的聪明 agent 更实用。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/01b431785dbe18ae.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/32c9c1189127520c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/5f56ff7d4266d9d9.png)

