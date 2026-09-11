---
title: tools.md 实践：把本地配置和环境差异写成 Agent 看得懂的事实清单
feedId: 37101
source: 综合讨论
publishedAt: 2026-09-11
---

## 背景

OpenClaw 的 workspace 里，`AGENTS.md` 定义行为约定，`tools.md` 负责另一件事：告诉 Agent **这台机器上的工具到底是什么状态**。它不是功能说明书——能力怎么调用是 skill 层的事；它是一份本机环境事实，每次会话都会进入上下文。

很多人把它当成一次性占位文件，写两行就丢在那。等 workspace 通过 git 同步到第二台、第三台机器时，问题就开始积累。

## 问题

Agent 的执行失败，很大一部分不是能力不够，而是环境认知错位：

- 同一份 workspace 跑在 macOS 笔记本和 Linux 服务器上，Python 解释器路径、包管理器、代理端口全不一样；
- Agent 记着上个会话验证过的命令，换台机器直接执行，路径不存在，然后开始自由发挥；
- 修正只存在于对话里，新会话又错一遍。

共同点是：环境事实没有被固化成 Agent 每次都能读到、且在当前机器上**仍然为真**的内容。

## 做法

**1. 分层：全局约定 + 机器差异。** 所有机器一致的内容（目录约定、安全红线）放全局段；每台机器单独一节，用 `## Machine: <名字>` 区分。全局段进 git，机器段放 `tools.local.md` 并加入 .gitignore——机器特有的配置和密钥天然不该同步。

**2. 只写事实，每条带验证命令。** 每条环境描述都配一条 Agent 可自查的命令：

```markdown
## Machine: lab-server
- Python: 用 conda env `agent`，不要用系统 python
  verify: conda run -n agent python --version
- 出网走本机 7890 代理，git/pip 需单独设置
  verify: curl -x http://127.0.0.1:7890 -sI https://example.com | head -1
```

verify 命令的价值在于：环境漂移时 Agent 能自己发现，而不是拿着过期信息继续跑。

**3. 控制篇幅。** 这个文件每次会话都占 token。取舍标准很简单：删掉这条，Agent 的行为会变吗？不会就删。

**4. 让 Agent 参与维护。** 新机器初始化时，让它读全局段，跑一遍所有 verify 命令，把失败项整理成问题清单；你补答案，它写回机器段。定期（比如每月）重复一次这个审计循环。

## 踩坑点

- **写密钥**。tools.md 会进上下文，等同于把 token 明文塞进每次会话。密钥走环境变量，文件里只写"从哪个变量读"。
- **写成教程**。背景解释、使用教程都不需要，Agent 要的是结论和验证方式。
- **和 skill 文档重复**。"怎么调用某工具"归 skill，"这台机器上它有什么不一样"才归 tools.md。
- **过期信息比没有更糟**。Agent 对文件内容的信任高于对你的口头纠正，陈旧的路径会让它反复撞墙。

## 可复用建议

- 固定小节：Python / Node / 网络 / 路径 / 硬件，每节不超过五行，每节带 verify；
- 把这套结构做成模板放进全局段，新机器只需填空；
- 修改后让 Agent 跑一个最小冒烟任务（比如"用本机 python 打印一行字"）确认端到端可用，而不是只检查文件本身。

## 总结

tools.md 的本质，是把"口头记忆"变成"机器可读、可验证的环境事实"。三个关键词：**分层**（全局与机器分离）、**可验证**（每条事实配自查命令）、**精简**（只留影响决策的内容）。做好这三点，同一份 workspace 无论同步到多少台机器，Agent 都知道脚该往哪踩。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/962e793d91a0c766.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/be0d82182f5640b5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/2a79031bf1cbfb4f.png)

