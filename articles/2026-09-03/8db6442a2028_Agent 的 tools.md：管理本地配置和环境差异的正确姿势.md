---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 35888
source: 综合讨论
publishedAt: 2026-09-03
---

## 背景

OpenClaw 这类常驻本地的 Agent，能力边界不只取决于模型和插件，还取决于宿主机：装了哪些 CLI、路径在哪、是 macOS 还是 Linux、哪些命令有坑。全局提示词（如 AGENTS.md）回答"Agent 是谁、守什么规矩"，但"这台机器长什么样"需要单独的落点。tools.md 的定位就是如此：一份写给 Agent 看的、描述当前环境事实的清单，随 workspace 一起注入上下文。

## 问题

没有 tools.md 或写法不对时，常见三类事故：

1. **幻觉调用**：Agent 猜测环境里有 ffmpeg / jq，实际没装，反复试错浪费轮次。
2. **平台踩雷**：同一套指令在 macOS 和 Linux 服务器间搬运，brew/apt、路径、`sed -i` 的差异直接导致命令失败。
3. **知识散落**：机器差异一会儿写死在全局 prompt，一会儿靠聊天口头补充，换台机器全作废，还把 token 持续消耗在与本机无关的描述上。

## 做法

核心原则一句话：**全局的进全局配置，本机的进 tools.md，机密的谁都不进。**

一个我实际在用的结构：

```markdown
# tools.md（每台机器一份，随 dotfiles 管理）
## 环境
- OS: macOS 15 / arm64；容器内为 Ubuntu 22.04
## 可用 CLI
- ffmpeg 7.1（/opt/homebrew/bin/ffmpeg）
- 无 jq，JSON 处理用 python3
## 已知坑
- sed -i 必须带 '' 后缀
- 长任务走 nohup + 日志文件，不要前台挂起
## 禁止事项
- 不动 /etc/hosts；不做全局 pip install
```

三个要点：

- **只写事实与约束，不写任务流程**。流程属于 AGENTS.md，两处重复是文件腐烂的开始。
- **负面清单比正面清单更有用**。"没有 jq"比"有 brew"更能省掉一轮试错。
- **控制在 60~80 行内**。它每次会话都被注入，每一行都是持续成本。

防漂移：环境会变，写个十行的检查脚本（探测关键 CLI 是否存在、版本是否匹配），挂进 cron，产生 diff 就提醒自己更新。更省事的做法是在新会话开头让 Agent 自己跑一遍探测命令，与文件做 diff，过期段落当场改掉。

## 踩坑点

- **把 key、token 写进去**。tools.md 会注入每次会话，等于明文广播。机密走环境变量或凭据管理器，文件里只写"密钥从 X 处读取"。
- **写成使用教程**。Agent 不需要被教怎么用 ffmpeg，只需要知道有没有、在哪、有什么限制。
- **多机共用一份**。差异文件一旦共用，要么变最大公约数而失效，要么塞满 if-else。一台机器一份，按主机名区分。
- **只增不删**。过期的坑比没有坑更危险——Agent 会一直绕一个早已不存在的坑。

## 可复用建议

- 模板固定四段：**环境 / 可用工具 / 已知坑 / 禁止事项**，新机器十分钟填完。
- 与 dotfiles 一起进版本管理，提交前跑一次密钥扫描（如 gitleaks）。
- 团队场景：共性部分抽成模板仓库，成员各自实例化，只共享脱敏后的 diff 作参考。

## 总结

tools.md 的价值不在文件本身，而在它逼你回答一个问题：这台机器和别的机器到底有什么不同。把环境事实从 prompt 和聊天记录里拆出来，写成短小诚实的清单，再配上防漂移检查，Agent 在任何新机器上都能少走弯路。这是把 Agent 从"演示可用"推进到"日常可靠"的成本最低的一步。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/39c59ef554bce341.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/c915e7ddbf4733aa.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/9c4af242e3dc8894.png)

