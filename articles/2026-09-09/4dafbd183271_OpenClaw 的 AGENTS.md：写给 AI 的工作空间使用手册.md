---
title: OpenClaw 的 AGENTS.md：写给 AI 的工作空间使用手册
feedId: 36667
source: 综合讨论
publishedAt: 2026-09-09
---

## 背景

OpenClaw 的 agent 每次进入一个工作空间，第一件事不是读代码，而是找上下文。README.md 是写给人的，讲"这个项目是什么"；而 agent 需要的是"在这个项目里怎么干活"：用什么包管理器、测试怎么跑、哪些目录不能动。AGENTS.md 就是补上这一层的约定——放在工作空间根目录、专门写给 AI 看的操作手册。OpenClaw 加载工作空间时会自动读取它，作为上下文注入给 agent。

## 问题

没有 AGENTS.md 时，这些场景很常见：

- 每次对话都要手动粘贴一遍构建命令和规范，忘了就翻车；
- agent 自作主张用 npm，而项目用的是 pnpm，装出两套 node_modules；
- 它不知道 `legacy/` 是只读目录，改完直接提了 PR；
- 团队里每个人的 prompt 模板各一份，修了 bug 修不到别人那里。

本质问题：上下文散落在对话里，没有沉淀到工作空间——不可版本化、不可复用、不可 review。

## 做法

从一个最小版本开始，建议控制在 60 行以内：

```markdown
# AGENTS.md

## 项目概览
- 单体仓库，核心逻辑在 core/，插件在 plugins/

## 常用命令
- 安装：pnpm install（不要用 npm/yarn）
- 测试：pnpm test --filter <pkg>
- 构建：pnpm build

## 代码约定
- 提交信息用 Conventional Commits
- 不要手动改 generated/ 下的文件

## 禁止事项
- 不要动 legacy/，只允许新增文件
- 不要升级 lockfile 中的依赖版本
```

三条写法经验：

1. **用祈使句，不用说明文。**"测试用 pnpm test"比"本项目使用 pnpm 作为包管理器，测试可通过 test 脚本执行"有效得多。
2. **分层放置。**根目录放全局约定，`plugins/` 等子目录放局部规则，OpenClaw 按就近原则合并生效。
3. **迭代维护。**agent 犯一次错，就把纠偏动作写进去。我维护了一份"翻车记录"，同一个坑踩两次，就升格为正式规则。

## 踩坑点

- **写成 README 的复制品。**两份文件内容重复甚至冲突时，agent 行为不稳定。README 讲 why，AGENTS.md 讲 how，各管各的。
- **太长。**超过两三百行后遵守率明显下降，token 成本也上去了。规则要像测试用例一样，只挑高价值的写。
- **规则互相矛盾。**"不要动 lockfile"和"依赖保持最新"放一起，agent 只能猜。发现冲突就合并或删除。
- **内容过期。**命令换了、目录重构了，AGENTS.md 还是旧的。把它的更新纳入 PR review：改了构建流程却不改 AGENTS.md，视同改漏。

## 可复用建议

- **当配置代码管理**：进版本库、走 review、有变更记录，别当成随手笔记。
- **拿不准写什么，直接问 agent**："你要在这个仓库跑测试，会执行什么命令？"它的回答暴露了它缺哪些上下文。
- **和 MCP 工具分工**：工具描述里写"怎么调用"，AGENTS.md 里写"什么时候该调用"，两边不重复。
- **模板化**：新项目从模板 fork，比每次从零写快得多，也保证团队间格式一致。

## 总结

AGENTS.md 是成本极低、复利很高的基础设施：一个文件，把散落在对话里的隐性上下文变成版本化的显性规则。它不智能，甚至有点笨——但 agent 恰恰需要这种明确、无歧义的手册。建议今天就从 20 行的最小版本写起，之后每次 agent 犯错，都是给它添一条规则的时机。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/eb2c7c44edb301ae.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/f041da543996f6fe.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/ec200c8644eaa10b.png)

