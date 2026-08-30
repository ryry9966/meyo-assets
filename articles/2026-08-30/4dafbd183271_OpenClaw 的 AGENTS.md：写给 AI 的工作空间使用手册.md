---
title: OpenClaw 的 AGENTS.md：写给 AI 的工作空间使用手册
feedId: 35343
source: 综合讨论
publishedAt: 2026-08-30
---

## 背景
在 OpenClaw 里接入 MCP、插件和多个 Agent 后，最常见的问题不是模型能力不足，而是它每次启动都像第一次进入这个仓库。你在对话里反复解释“别碰 docs/ 下的自动生成文件”“部署要用 make release 而不是 npm run build”，但这些上下文只存在于临时聊天记录里，换一个会话就消失。

## 问题
没有统一入口，规则散落在 README、issue 和口口相传中，导致不同 Agent 行为不一致：同一个“禁止修改数据库迁移脚本”，A 严格遵守，B 却直接重写。更麻烦的是，当 Agent 通过 MCP 拿到文件系统或终端工具时，越界操作往往发生在它“不知道规则”的情况下，而不是故意违规。

## 做法/步骤
在项目根目录建立 AGENTS.md，并纳入版本控制。文件内容以“给 AI 的操作手册”为目标，建议包含以下骨架：

- 项目用途：一两句话说清这个仓库是干什么的。
- 常用命令：列出构建、测试、运行、部署命令，精确到参数。
- 目录约定：哪些目录是源码、哪些是生成物、哪些禁止手动编辑。
- 禁区：明确列出不可执行的操作，例如“不要删除 .lock 文件”“不要修改 tests/fixtures/ 下的文件”。
- 工具与插件规则：说明 MCP 工具的使用边界，如“只允许使用 github MCP 读取 issue，不允许创建 PR”。
- 验证方法：告诉 Agent 改完代码后如何自测，比如跑哪条命令、看哪个输出。

写法上注意：多用祈使句，每条规则单独一行，优先给示例而不是解释。比如写“不要在根目录创建临时脚本；统一放在 scripts/tmp/”，而不是“请保持根目录整洁”。

文件保存后，在 OpenClaw 会话中显式引用或配置为启动时自动加载。建议第一次让 Agent 读完后复述三条关键规则，验证它是否真的解析到了。

## 踩坑点
1. 文件太长：超过 200 行后，模型容易忽略靠后的内容，也浪费上下文。控制在 80–150 行比较合适。
2. 规则模糊：像“注意安全”这种话等于没说。要写成“禁止在 AGENTS.md 中写入任何密钥或 token”。
3. 与系统 prompt 冲突：如果 OpenClaw 默认行为是“自动提交代码”，而 AGENTS.md 写“不要自动 git commit”，Agent 可能选择性执行。需要测试实际行为。
4. 只写不维护：项目结构变了，AGENTS.md 却还是旧路径，反而误导 Agent。

## 可复用建议
- 保持短小，超出就拆分为 `AGENTS-build.md`、`AGENTS-deploy.md` 等，主文件只做索引。
- 按场景分节：日常开发、发布、排障，减少 Agent 需要加载的无关规则。
- 使用“必须/禁止/优先”等词明确强度，避免“建议”和“必须”混在一起。
- 把 AGENTS.md 当作工作空间的长期资产，而不是一次性 prompt。每次目录或命令变更时同步更新。
- 可以写一个几行的脚本，在 Agent 启动时输出 AGENTS.md 摘要，作为上下文的固定开头。

## 总结
AGENTS.md 的价值不是给 AI 套上枷锁，而是把重复的解释成本降到最低，让 Agent 在正确的边界内干活。一个维护良好的 AGENTS.md，能显著减少多 Agent 协作中的误操作和返工，是 OpenClaw 工程化实践里值得投入的一部分。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/9032132739dd9c79.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/aaeb82d3fdab1815.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/fbee5c149ce99994.png)

