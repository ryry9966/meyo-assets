---
title: OpenClaw 的 AGENTS.md：写给 AI 的工作空间使用手册
feedId: 37222
source: 综合讨论
publishedAt: 2026-09-12
---

## 背景

AGENTS.md 是近一年逐渐成形的一个社区约定：在仓库根目录放一个纯 Markdown 文件，专门写给 AI 编码/自动化 Agent 看。一句话概括——人读 README，Agent 读 AGENTS.md。OpenClaw 的 Agent 进入工作空间时会优先读取它，把它当作这个项目的“上岗须知”。目前主流的编码类 Agent（CLI Agent、IDE 内置 Agent 等）基本都认这个约定，MCP 工具调用之外的环境上下文，主要就靠它承载。

## 问题

没有 AGENTS.md 时，Agent 每次进场都在猜：

- 装依赖用 npm 还是 pnpm？测试命令到底是什么？
- 项目有哪些不成文规范：错误处理方式、目录归属、哪些文件不许碰？
- “做完了”的标准是什么——跑过 lint？跑过测试？跑过哪个脚本？

结果是每个会话都要人肉贴上下文，团队成员贴的还各不相同，Agent 行为不可复现，同样的错反复犯。这些问题不解决，自动化规模越大，浪费越多。

## 做法

1. **建文件**：仓库根目录放 `AGENTS.md`；monorepo 可在子目录再放一份，离代码近的生效，形成分层覆盖。
2. **按优先级写内容**：
   - 一段话项目概述，不超过五行；
   - 精确命令：安装、构建、测试、lint，直接给可执行命令，不要写“确保测试通过”这种描述；
   - 硬约束：哪些目录只读、迁移怎么处理、禁止引入的依赖；
   - 目录导览：一两行说清新手最容易迷路的部分；
   - 完成标准：声明完成前必须跑通的验证步骤。
3. **控制篇幅**：建议 100 行以内。Agent 的注意力是稀缺资源，写得多不如写得准。
4. **纳入版本控制**：像代码一样走 PR 评审，规则变更有据可查。
5. **建立迭代机制**：Agent 同一个错犯两次，就把对应规则写进去。

骨架示例：

```markdown
# Project
一段话描述。

## Commands
- Install: pnpm install
- Test: pnpm test --filter core
- Lint: pnpm lint

## Conventions
- 错误统一用 Result 封装，不要裸 throw
- packages/legacy 只读，改动需人工确认

## Definition of done
lint + test 全绿，并运行 scripts/verify.sh
```

## 踩坑点

- **过期内容比没有更糟**。命令改了没同步，Agent 会拿着错误指令反复失败。重构后记得更新，最好和 CI 共用同一组命令。
- **写成作文**。大段散文 Agent 抓不住重点，规则要短、可执行、可验证。
- **层级矛盾**。根目录和子目录的规则互相冲突时，Agent 行为会变得随机，修改前先通读一遍层级关系。
- **写入敏感信息**。这个文件在 git 里，放 token 等于公开。
- **当成提示词堆放处**。AGENTS.md 是稳定手册，一次性任务需求走会话内提示，别污染公共入口。
- **指望它兜底安全**。它是约定不是沙箱，权限控制要靠工具层（hooks、审批、隔离环境）来保证。

## 可复用建议

- 每条指令都写成可执行/可验证的形式：命令 > 描述。
- 维护一个“踩坑记录”区，反复出现的问题先记录，稳定后再固化为规则。
- 命令与 CI 管线对齐，让 Agent 的本地验证和流水线验证是同一套。
- 大型 monorepo 按子项目拆多个 AGENTS.md，各管各的。
- 新人入职也能直接读它——写给 Agent 的清晰规则对人同样有效，这本身就是一次文档质量检查。

## 总结

AGENTS.md 本质上是“配置即文档”：把散落在聊天记录和 README 角落里的工程约定，收敛成一个 Agent 可稳定消费的单一入口。维护成本很低，换来的是跨会话、跨成员、跨工具的一致行为。如果你的 OpenClaw 工作空间还没有这份手册，今天花二十分钟写一版，之后每次 Agent 犯错就补一条——它会比你想象中更快变得可靠。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/ad79b78e72fd21bc.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/98f258b7fe922ca8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/b47d698d49c678df.png)

