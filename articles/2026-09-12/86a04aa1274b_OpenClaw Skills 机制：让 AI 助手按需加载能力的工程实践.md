---
title: OpenClaw Skills 机制：让 AI 助手按需加载能力的工程实践
feedId: 37113
source: 综合讨论
publishedAt: 2026-09-12
---

# 背景

跑 OpenClaw 久了都会遇到同一个问题：能力越多，系统提示词越肥。早期我把常用流程（发版检查、日志排查、消息模板）全塞进 AGENTS.md，结果常驻上下文多出几千 token，模型反而开始漏看关键指令。

Skills 机制就是为了解决这个矛盾：每个技能是一个独立的 `SKILL.md`，会话启动时只把「技能名 + 一句 description」编译进系统提示词，正文在模型真正决定使用该技能时才被读入。这就是渐进式披露（progressive disclosure）——代价是把"何时用什么"的判断交给模型，收益是常驻上下文几乎恒定。

# 问题

实际使用中有三类典型失败：

1. 技能写了但从不被触发——description 没说清"什么时候用"，模型无从路由；
2. 触发了但正文过长，等效于把 prompt 重新塞肥；
3. 依赖缺失（缺二进制、缺环境变量）时技能被静默隐藏，用户以为模型"不会"，其实是技能根本没装载。

# 做法

以一个"发布前检查"技能为例，先建目录：

```bash
mkdir -p ~/.openclaw/workspace/skills/release-check
```

`SKILL.md` 分两半——frontmatter 负责路由，正文负责执行：

```markdown
---
name: release-check
description: 当用户要求发版、打 tag 或部署前检查时使用。执行依赖确认、变更清单和回滚预案检查。
metadata:
  openclaw:
    requires:
      bins: ["git", "gh"]
---

## 步骤
1. `git status` 确认工作区干净
2. 用 `gh` 检查未关闭的 issue 和 PR
## 禁止
- 不要直接 push main
```

三个要点：

- **description 是路由依据**。模型只靠这一句决定是否读正文，写成"当……时使用"的触发条件加一句能力概述即可，别写成自我介绍。
- **用 `requires` 声明依赖**。环境不满足时技能会在清单中标记不可用或隐藏，避免模型拿着跑不通的流程硬执行。
- **正文写操作手册，不写背景故事**。命令、顺序、边界条件、禁止事项，几百行以内。

验证用 `openclaw skills list` 看是否被识别、依赖是否满足，`openclaw skills info <name>` 看详情。注意：技能清单在会话创建时编译，改完必须新开会话才生效。

# 踩坑点

- 改了 SKILL.md 但老会话没反应——不是缓存 bug，是清单在会话启动时注入。
- 优先级是 workspace > 用户级 > 内置技能，同名会覆盖。定制内置技能前先确认自己在改哪一层。
- **Skills 不是 MCP**。技能教模型"怎么用好已有的工具"（exec、browser、消息接口），不能注册新的工具端点；需要新接口时老实上 MCP server。两者互补而非互替。
- 一个技能只做一件事。我早期写过 400 行的"运维大全"，触发率很高，但模型执行到一半开始自由发挥；拆成三个小技能后稳定很多。

# 可复用建议

- 固化 description 模板：「当 \<触发场景\> 时使用。\<一句话能力\>」。
- 把团队 SOP 逐步沉淀为技能，纳入 git 管理。评审 description 就是在评审能力边界，成本很低。
- 定期用 `skills list` 审计：长期不触发的技能，多半是 description 写得不好，或者它本来就不该是技能。

# 总结

Skills 的本质是给上下文做预算管理：常驻区只放索引，正文按需加载。它不增加模型能力，只是让已有能力在正确的时机被正确读到。实践中，写好一句 description，比写十页正文更能决定一个技能的命运。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/f725d2c652e47f06.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/f9cd349a80d5db70.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/dab897161ba5e762.png)

