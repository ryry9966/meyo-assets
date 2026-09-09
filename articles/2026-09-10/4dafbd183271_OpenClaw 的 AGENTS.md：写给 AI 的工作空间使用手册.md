---
title: OpenClaw 的 AGENTS.md：写给 AI 的工作空间使用手册
feedId: 36822
source: 综合讨论
publishedAt: 2026-09-10
---

## 背景

用 OpenClaw 跑自动化任务的同学大概都有类似体会：agent 的实际表现，往往不取决于模型本身，而取决于它对工作空间的了解程度。README 是写给人看的，讲"这是什么"；agent 需要知道的是"在这里该怎么干活"——用什么包管理器、测试怎么跑、哪些目录不能碰。AGENTS.md 就是补上这一层的约定：一份放在工作空间根目录的纯 Markdown 文件，OpenClaw 在每次会话启动时自动读取，作为 agent 的默认操作上下文。

## 没有它会发生什么

几个社区里反复出现的真实场景：

- agent 用 npm 装依赖，项目实际用 pnpm，lockfile 被污染；
- 改完代码不跑测试——不知道测试命令，或者一口气跑了全量用例拖十几分钟；
- 动了 `generated/` 下的文件，下次代码生成时本地改动全部冲突；
- 每个新会话都要在对话里重新交代一遍规矩，prompt 越写越长，还容易漏。

共性很明显：约定只存在于人的脑子里或聊天记录里，没有沉淀成 agent 能稳定读取的文件。

## 怎么写

在仓库根目录建 `AGENTS.md`，建议分五段：

```markdown
# 项目概览
一两句话说清这是什么项目，写给 agent 看。

# 常用命令
- 安装依赖：pnpm install
- 本地验证：pnpm lint && pnpm test --changed
- 构建检查：pnpm build

# 代码约定
- TypeScript strict 模式，禁止 any
- 提交信息遵循 Conventional Commits

# 禁止事项
- 不要修改 generated/ 与 vendor/ 下任何文件
- 不要升级依赖版本，除非任务明确要求

# 完成标准
- 改动必须通过 lint 与相关单测
- 涉及接口变更时，同步更新对应 MCP tool 描述
```

Monorepo 可以在子目录再放一份更细的 AGENTS.md，agent 进入该目录工作时以就近文件为准。做插件或自动化流程的，把本地联调命令和插件加载方式也写进去，省掉大量重复解释。

## 踩坑点

1. **写太长。** AGENTS.md 会占上下文，超过一两百行后，模型对靠后内容的遵循度明显下降。原则：每一行都应该能改变 agent 的某个行为，做不到就删。
2. **写成项目愿景。** "致力于打造高质量xxx"这类话对 agent 没有操作意义，等于浪费 token。
3. **内容漂移。** 命令改名、目录重构后忘了更新，agent 会反复失败。建议在 CI 加一个轻量 job，把 AGENTS.md 里的命令原样跑一遍。
4. **写入敏感信息。** 这份文件会进入模型上下文，任何 key、内网地址都不要放。
5. **与口头指令冲突。** 对话里说"这次可以动 generated/"，文件里却禁止，agent 行为会不稳定。临时例外在对话里说清原因，长期变更改文件本身。

## 可复用的习惯

- 把 AGENTS.md 当代码管理：进 git、走 review，谁改了构建流程谁更新它。
- agent 每犯一次错，优先转化为一条 AGENTS.md 规则，而不是下次口头提醒——这是把调试成本变成资产。
- 定期做减法：删掉三个月没"拦住"过任何错误的条目。

## 总结

AGENTS.md 的价值不在格式，而在把团队"怎么在这个仓库干活"的隐性共识显式化，并以 agent 稳定可读的方式注入每次会话。写它的过程，本质是在梳理你自己工程的规矩。建议从今天起，把你最近三次在对话里向 agent 重复的解释搬进 AGENTS.md——那通常就是它最该有的前三行。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/a7eb411847ba587c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/31f0eff076dbdf0c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/26315b5b001caa9f.png)

