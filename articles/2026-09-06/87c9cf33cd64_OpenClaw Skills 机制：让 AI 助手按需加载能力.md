---
title: OpenClaw Skills 机制：让 AI 助手按需加载能力
feedId: 36289
source: 综合讨论
publishedAt: 2026-09-06
---

## 背景

做 Agent 的老矛盾：能力越多，上下文越胖。把几十个集成的用法全塞进 system prompt，token 成本高，模型反而抓不住重点；不塞，模型又不知道自己“会什么”。OpenClaw 的 Skills 机制走的是渐进式披露路线：每个技能常驻上下文的只有名字和一段 description，具体的操作正文只在模型判断相关时才读入。相当于给助手一本按需翻阅的操作手册，而不是让它背下所有说明书。

## 问题

实际用下来，多数团队卡在三件事上：技能写了模型却不在对的时机调用；环境缺依赖时模型照样“自信地”执行；技能正文越写越长，按需加载退化成了变相全量加载。

## 做法

**1. 目录与结构。** 技能放在 `~/.openclaw/skills/<name>/SKILL.md`（工作区 `skills/` 目录同名时优先）。文件 = YAML frontmatter + Markdown 正文：

```markdown
---
name: invoice-export
description: 当用户要求导出当月发票或对账单为 PDF 时使用，依赖本地 invoice-cli。
metadata:
  openclaw:
    requires:
      bins: ["invoice-cli"]
      env: ["INVOICE_TOKEN"]
---
## 步骤
1. 先跑 `invoice-cli list` 确认数据源可用
2. 导出后核对文件路径再回复用户
```

**2. description 按“检索”来写。** 它是模型决定是否读正文的唯一依据，写成触发条件句，不是功能宣传语。

**3. 用 requires 做环境门控。** `bins`/`env` 不满足时技能标记为不可用，从源头避免幻觉调用。

**4. 验证与测试。** `openclaw skills list` 看是否被发现、是否 eligible；`openclaw skills info <name>` 看详情。然后直接问 agent 一个应触发该技能的问题，观察日志里它有没有读取 SKILL.md 正文。

**5. 按需启停。** 配置里按 agent 维度 enable/disable，不用的直接关——常驻的 description 也是成本。

## 踩坑点

- **description 写成口号**（“强大的发票工具！”）→ 模型不知道什么时候用。改成“当用户要求 X 时使用”。
- **正文塞成百科** → 等于全量加载。控制在一两百行，细节放脚本文件，SKILL.md 只写调用方式。
- **漏配 requires** → 缺 CLI 的机器上模型照样编命令，报错还很难排查。
- **混淆技能与工具**。Skills 是指令层，真正执行靠 CLI/MCP。不要在 SKILL.md 里虚构不存在的 API；调用不到的东西，就写明“提示用户先安装 X”。
- **同名冲突**。workspace 与全局目录重名时注意加载优先级，排查第一步永远是看 `skills list` 显示的来源。

## 可复用建议

一个技能一个场景，宁可拆细；description 不超过两句，正文控制在 300 行以内；可执行逻辑抽成脚本随技能目录一起做版本控制；定期审计技能列表，下线不再用的——手册页数本身也是开销。

## 总结

Skills 机制解决的是“知道”与“会做”的分离：常驻的只是索引，知识按需进入上下文。写好一个技能的关键不在内容多，而在 description 的触发描述准不准、正文是否克制。配合 MCP/CLI 做执行层，Skills 负责“什么时候、按什么步骤做”——技能数量一多，这套分工的收益会非常直观。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/8d5d4b58a8c3621e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/db80d85898f10db4.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/6ab6542761d48288.png)

