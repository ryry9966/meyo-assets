---
title: OpenClaw Skills 机制：让助手按需加载能力，而不是全塞进上下文
feedId: 36037
source: 综合讨论
publishedAt: 2026-09-04
---

## 背景

在 OpenClaw 的能力体系里，MCP 工具和 Skills 是两回事：工具解决"能不能调接口"，Skill 解决"知道该怎么做"。一个 Skill 本质上就是一个目录加一个 `SKILL.md`：frontmatter 里写 `name` 和 `description`，正文写操作指引。

运行时的关键设计是：OpenClaw 只把所有 Skill 的**名称和描述**注入系统提示，形成一份轻量索引。Agent 判断当前任务命中某个 Skill 后，再用 read 工具把完整的 `SKILL.md` 读进来照做。这就是"按需加载"的核心——常驻上下文的只有索引，重型内容按需读取。

## 问题

不理解这个机制，通常有两类浪费：

1. 把所有文档、SOP、工作流全塞进系统提示或全局记忆文件，上下文膨胀，token 成本高，指令之间还互相干扰；
2. 装了一堆第三方 Skill，但助手从来不触发，或者乱触发——因为真正决定路由的，就是 `description` 那一行字。

## 做法

1. **建目录**：个人 Skill 放 `~/.openclaw/workspace/skills/<skill-name>/SKILL.md`，团队通用的放进版本库一起管理。
2. **声明依赖**：有外部依赖的用 `metadata.requires` 写清 `env` / `bins`，加载器会先做可用性检查，缺依赖直接标记不可用，避免运行时踩雷。
3. **正文用指令式写法**："当用户要求 X 时，先执行 Y，再 Z；若失败，回退到 W。"正文是 prompt 素材，不是 README，别写成产品介绍。
4. **大文档拆出去**：`SKILL.md` 只留判断逻辑和入口，长参数表、详细清单指向同目录的 reference 文件，让 Agent 真正需要时再读。
5. **验证**：`openclaw skills list` 看加载状态和依赖门槛，`openclaw skills info <name>` 看细节；然后开一个真实会话，从日志确认 Agent 是否真的 read 了你的 SKILL.md。

## 踩坑点

- **description 写得太泛**（"帮你处理文件"）永远轮不到它被触发；太宽（"处理一切任务"）则会抢路由。正确写法是描述触发条件："当用户要求导出周报或操作 git 仓库时使用"。
- **列表里可见 ≠ 能跑**：env 是在 gateway 进程里读的，改了 `.env` 或 shell profile 后要重启 daemon 才生效。
- **索引本身也吃 token**：几十个第三方 Skill 全装上，光索引就不便宜，定期清理不用的。
- **命令必须真实可执行**：Agent 不会猜你们内部封装的脚本，路径、参数、失败分支都要写死在正文里。
- **别和工具重复**：已有现成 MCP tool 的，Skill 只写编排逻辑，不要再造一份工具描述。
- **注意命名冲突**：自定义 Skill 和内置 Skill 同名时，行为可能和你预期不一致。

## 可复用建议

- 一个 Skill 只做一件事，复杂流程拆成多个 Skill，或让一个 Skill 引导调用多个工具。
- 把 skills 目录纳入 git，像管代码一样 review、打版本、写变更说明。
- 把路由（description）和执行（正文）当成两个独立接口分别打磨；改 description 的收益通常最大。
- 上线前用一条真实任务做回归：是否被触发、是否读文件、执行是否成功，三步都过再留。

## 总结

Skills 的价值不在"装得多"，而在"索引轻、命中准、内容按需"。理解了"描述做路由、正文做执行、requires 做前置检查"这三层，就能让助手在保持上下文干净的同时，随时拉起成套的操作能力——这也是它和"把文档全塞进 prompt"最本质的区别。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/eedc84c1269892e1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/450b03e7827505d5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/42f66e8ed88ab0f2.png)

