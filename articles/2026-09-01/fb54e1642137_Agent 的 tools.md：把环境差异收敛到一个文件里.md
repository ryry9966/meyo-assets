---
title: Agent 的 tools.md：把环境差异收敛到一个文件里
feedId: 35715
source: 综合讨论
publishedAt: 2026-09-01
---

## 背景

跑本地 Agent 一段时间后你会发现，最常坏的不是模型，而是环境。同一套提示词和技能，在办公机上跑得好好的，回到家里的 NUC 就报 `command not found`；同事拿去复现，第一件事是改路径。原因很简单：Agent 的"世界观"里写死了你这台机器的事实——Python 用 uv 还是 conda、代理怎么走、有没有 GPU、MCP server 从哪启动。

## 问题

这些环境事实目前通常散落在三处：系统提示词、各技能里的硬编码路径、以及你自己的脑子。前两处随机器漂移，第三处不可迁移。插件和 MCP 一多，工具名、启动方式、依赖版本各自为政，Agent 只能靠猜，猜错就编造一个不存在的命令。

## 做法

我的方案是把机器相关的事实收敛进一个 `tools.md`，放在 OpenClaw 工作区根目录，随工作区文件一起加载，不用改任何代码。核心原则只有一条：**每条事实必须自带验证命令**。

```markdown
## runtime
- python: /usr/bin/python3 (3.12), verify: python3 --version
- pkg: uv preferred, pip fallback
## paths
- workspace: ~/agents/ws
## constraints
- no GPU, CPU-only inference
- proxy: http://127.0.0.1:7890 for pip/npm
## forbidden
- 不要改 /etc/hosts
```

三条配套规则：

1. **分层**。仓库只提交 `tools.example.md`，真实 `tools.md` 进 `.gitignore`。项目级差异放项目目录下另一份，后加载者覆盖先加载者。
2. **可再生**。写个十行脚本扫描 `which python3 / node / docker` 等输出重写基础段，手写段用注释标记、重生成时保留。文件头加生成时间，超过两周未更新视为可疑。
3. **引用密钥而非内联**。只写"token 位于 `~/.config/xx/token`，经环境变量注入"，绝不写值。

## 踩坑点

- **文件腐烂**。写完就忘，三个月后文件里是 Python 3.10，实际已是 3.13，Agent 据此的判断全错。过期信息比没有信息更糟，所以再生成脚本不是可选项。
- **写太长**。它每次都进上下文，我压在 60 行内。用不上的条目删掉，Agent 真需要时会自己跑命令确认。
- **被同步工具覆盖**。dotfiles 一同步，A 机的 `tools.md` 抹到了 B 机。务必排除在同步之外，或干脆开机重算。
- **盲信文件**。即使是自己写的，Agent 执行关键操作前也应先跑 verify。我在系统提示里加了一句："环境事实以实测为准，tools.md 仅为提示。"
- **跨平台差异**。Windows 下 `~/` 和引号行为不同，条目直接写双平台写法，别让 Agent 自行发挥。

## 可复用建议

- 把"事实 + verify 命令"当固定格式，缺 verify 的条目直接删。
- 团队场景下，`tools.example.md` 就是最好的 onboarding 文档，新人复制改名即可跑通。
- MCP server 的启动参数、端口这类易变信息也收进来，避免散落在多个 JSON 里。

## 总结

`tools.md` 解决的不是技术难题，而是一致性问题：让 Agent 对"这台机器"的认知有一个单一事实来源，且可验证、可再生、可分层。它很朴素，但把环境漂移这类最烦人的隐性故障变成了显式文件。别急着设计完美模板，先列出你机器上 Agent 最常用的五个工具，就是最好的起点。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/714e7322ef4f343c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/8bc6ce186a2363dd.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/6bde78c4149c9975.png)

