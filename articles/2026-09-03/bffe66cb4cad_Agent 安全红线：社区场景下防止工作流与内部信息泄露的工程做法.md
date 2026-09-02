---
title: Agent 安全红线：社区场景下防止工作流与内部信息泄露的工程做法
feedId: 35860
source: 综合讨论
publishedAt: 2026-09-03
---

## 背景

跑 OpenClaw 的人多多少少有几套自动化流程：定时整理 issue、挂几个 MCP server 读代码库和运维面板。用顺手之后，很容易让同一个 agent 兼任"社区答疑"角色——在群里、论坛里帮人排查问题。隐患就从这里开始：这个 agent 的上下文里，混着你不想公开的一切。

## 问题在哪

泄露通常不是 agent "主动泄密"，而是三条被动路径：

1. **上下文携带**：内部会话历史、memory、system prompt 里的流程细节，在社区对话中被复述出来。
2. **工具返回值**：MCP 工具报错时把内网地址、绝对路径、堆栈原样吐给模型，模型再转述给提问者。
3. **人工分享**：你贴日志、截图、配置片段求助时，token 和 session id 就在里面。

提示词里写一句"不要泄露内部信息"是不够的，长对话和诱导之下模型并不可靠。红线要靠工程手段兜底。

## 做法

**1. 社区场景用独立 profile。** 不复用生产 agent 的 workspace，单独建目录，只放公开文档；system prompt 只写公开知识边界，内部流程细节一个字不放。

**2. 工具白名单收到最小集。** 对外 agent 禁用全盘文件读写和 shell，只挂只读 MCP 实例、指向 `docs/` 的检索工具。原则：它能触达的数据面 = 你愿意公开的最大范围。

**3. 记忆隔离。** 社区 profile 关闭长期记忆，或指向独立 memory store。我们踩过共享 memory 导致社区 agent 说出内部排期的坑——多 agent 共库是高发事故源。

**4. 输出审查 hook。** 消息发出前加一层正则检查：密钥格式（`sk-`、`AKIA` 前缀）、内网域名/IP 段、内部路径前缀，命中即拦截替换为占位符。OpenClaw 插件体系里做个 pre-send hook 很轻量。

**5. 日志脱敏脚本。** 凡是要贴出去的日志先过脚本：env 值、URL query 里的 token、绝对路径、邮箱统一替换。脚本进 git，别靠肉眼。

## 踩坑点

- 以为"不给工具就安全"。上下文里已有的内容照样被复述，memory 注入是重灾区。
- MCP 错误信息默认带 host 和路径，只读不等于无泄露。
- 社区帖子本身可能带 prompt injection——agent 读帖后可能被诱导"打印配置"。对外 agent 的一切输入源都要当不可信处理。
- 截图排查时漏看角落里的 session id 和二维码。

## 可复用建议

- 维护一份红线清单：密钥、内网拓扑、客户数据、未发布规划。接入任何新 MCP/插件前对照三问——它能读什么？输出进谁的上下文？报错返回什么？
- pre-send hook + 脱敏脚本做成模板，新 profile 直接套用。
- 每月做一次红队演练：让同事专门诱导社区 agent 泄露，失败案例回填清单。

## 总结

Agent 安全不是一句提示词的事，而是三层兜底：**权限面收窄、上下文隔离、输出审查**。把"社区 agent 默认不可信"当成工程前提，红线才守得住。欢迎在回帖里补充你们遇到的真实泄露案例和拦截规则。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/f092cbc66dc25c28.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/42444b7db9e1181d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/997f92dfb25e502a.png)

