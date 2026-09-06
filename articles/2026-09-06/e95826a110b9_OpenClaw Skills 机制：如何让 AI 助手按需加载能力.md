---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 36304
source: 综合讨论
publishedAt: 2026-09-06
---

## 背景

OpenClaw 的 agent 能力不是硬编码的，而是通过 Skills 机制扩展：每个 skill 是一个文件夹，核心是一个 `SKILL.md`——frontmatter 里写 `name` 和 `description`，正文是给模型看的操作说明，旁边可以带脚本和参考文档。关键设计是**渐进式加载**：系统提示里只注入每个 skill 的名称和一句话描述，正文只在模型判断相关时才读进来。

## 问题

不理解这个机制，通常走向两个极端：

- 把所有能力说明都塞进系统提示，几万 token 常驻上下文，响应慢、成本高，注意力还被稀释；
- 或者 skill 描述写得太模糊，该触发的时候模型根本想不起来用。

Skills 本质上是在解决"能力越加越多"和"上下文有限"之间的矛盾：常驻的只是索引，正文按需取。

## 做法

1. **建目录**。用户级 skill 放 `~/.openclaw/skills/`，工作区 skill 放 workspace 下的 `skills/`，一个 skill 一个文件夹。内置 skill 遵循同样结构，可以直接当参考模板。
2. **写 SKILL.md**，frontmatter 两行：`name`（小写连字符，与目录名一致）、`description`（触发条件）。
3. **description 按"何时用"写，不按"是什么"写**。"当用户要求生成周报、汇总本周 git 提交时使用"，远比"周报助手"有效——这是模型判断是否加载的唯一依据。
4. **正文控制篇幅**：只写模型不知道的东西——内部流程、命令、约定。通用常识不要写。
5. **长流程外移**：详细文档放 `references/`，可执行脚本放 `scripts/`，正文只写"什么情况下读哪个文件、怎么调脚本"。
6. **声明依赖**：需要 API key 的用 requires 标注环境变量，缺依赖时 skill 会被标记为不可用，而不是运行到一半才炸。也可在 openclaw.json 中按名称手动开关。
7. **验证**：`openclaw skills list` 看加载状态，`openclaw skills info <name>` 看详情；重载会话后用自然语言试触发，观察是否命中。

一个最小示例：

```markdown
---
name: weekly-report
description: 当用户要求生成周报、汇总本周 git 提交或整理项目进展时使用
---
1. 运行 scripts/collect.py 获取本周提交
2. 按 references/template.md 的结构输出
```

## 踩坑点

- **description 写成功能简介**，模型无法判断加载时机，表现为"从来不触发"或"乱触发"。
- **SKILL.md 动辄几千字**，每触发一次吃一次 context，渐进式加载形同虚设。细节全部外移。
- **目录名与 name 不一致**、大小写混用，导致加载异常。
- **相对路径坑**：脚本里用相对路径，工作目录一变就找不到，用绝对路径或基于 skill 根目录拼接。
- **两个 skill 描述重叠**会互相抢触发，保留一个或收紧措辞。
- **改完不重载会话**，误以为已生效，排查半天其实是旧缓存。

## 可复用建议

- 粒度按"任务动词"切：发布、巡检、备份，一个 skill 只管一类事。
- 能脚本化的别写进提示词——skill 只教模型怎么调脚本，确定性交给代码。
- skills 目录放进 git，description 的改动走 PR 评审：它直接决定触发率，值得被 review。
- 定期跑 `openclaw skills list`，清理长期不触发或一直缺依赖的 skill。

## 总结

Skills 的价值不在"能挂多少能力"，而在"上下文里只留索引、用时才取正文"这个取舍。把 description 当检索关键词去打磨，把正文当内部文档去维护，skill 才能稳定、低成本地触发。这是机制层面的功课，比堆提示词划算得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/9e60a62db4ee9e09.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/559db492217c50fd.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/93ee7402ce744d60.png)

