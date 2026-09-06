---
title: Agent 的 tools.md：管好本地配置与环境差异的务实做法
feedId: 36293
source: 综合讨论
publishedAt: 2026-09-06
---

## 背景

OpenClaw 的 workspace 里有几个约定文件，Agent 每次会话都会读取：AGENTS.md 约束行为，MEMORY.md 存记忆，而 tools.md 的角色很特殊——它是 Agent 认识「这台机器」的主要入口。你不在场的时候，它对本地环境的全部了解基本来自这份文件。

我的 Agent 跑在三类机器上：日常的 macOS 笔记本、一台 Linux 工作站、一台海外 VPS。三台机器的包管理器、Python 版本、代理设置、可用工具完全不同。这个问题不解决，Agent 到哪台机器都在重新踩坑。

## 问题

不写 tools.md，Agent 只能猜：猜路径、猜包管理器、猜 `python3` 指向哪个解释器。猜错了就装错依赖、写错脚本，你得在对话里一次次纠正。

但乱写更糟。常见反面写法有三种：把三台机器的说明全堆进同一份文件；整段粘贴 CLI 的 help 输出；把 token 和内网地址直接写进去。结果要么指令互相矛盾，要么上下文被垃圾信息撑爆，要么泄漏凭据。

## 做法

核心原则一句话：**只写与 Agent 默认假设不同的东西**。它默认会 `ls`、会 `git`，这些不用教；它不知道你机器上 `python3` 实际指向 conda 的 3.10，这才要写。

推荐结构（按需删减）：

```markdown
# 环境
- macOS 14, Apple Silicon; brew 位于 /opt/homebrew/bin
- python3 → pyenv 3.12.x，系统 python 不要用
# 网络
- 默认走代理 http://127.0.0.1:7890，内网 git 例外
# 已知的坑
- ffmpeg 是 6.x，部分参数与网上教程不一致
- /data 是外挂盘，重启后需手动挂载
# 禁区
- 不要动 ~/.ssh 和 crontab
```

三条工程实践：

1. **一台机器一份文件**。tools.md 跟机器走，不跟项目走。用私有 dotfiles 仓库管理，每台机器只部署自己那份；公共模板单独维护，改完模板再各机差异化。
2. **内容最小化**。Agent 会自己跑 `--help`、自己探路，tools.md 只写「哪里有坑」和「哪里不许去」。
3. **定期校准**。每两周让 Agent 对比文件与实际环境（列出版本、检查路径是否存在），人审后合入。装了新工具、换了代理，当天就更新。

## 踩坑点

- **凭据入文件**：token 会随上下文进入模型。写「token 在 `~/.config/xx`，通过环境变量读取」这种指路即可，永远不写值。
- **过期比不写更糟**：文件里写了不存在的路径，Agent 会反复尝试甚至脑补。给文件加一行「最后校准日期」。
- **大段粘贴 help/文档**：纯浪费上下文，Agent 自己查更快。
- **写了但没生效**：确认 workspace 路径配置和文件名大小写与约定一致，否则等于没写。
- **与 MCP 配置重复**：MCP server 的启停放配置文件，tools.md 只写「这些 server 用起来要注意什么」，别维护两份真相。

## 可复用建议

- 模板六节：系统与 shell / 语言与版本管理 / 网络 / 路径约定 / 已知的坑 / 禁区。
- 「例外才写」：默认假设一律不写，这是文件不腐烂的关键。
- 把维护外包给 Agent：让它定期 diff 环境与文件、输出增删建议，人只做审核。
- 敏感信息分层：凭据走环境变量或密钥管理，tools.md 只留指路。

## 总结

tools.md 的本质，是把「环境知识」从一次性对话里抽出来，固化成机器级的持久上下文。写少、写准、跟机器走、定期校准——做到这四条，Agent 在你的每台机器上都能少走弯路，你的纠错成本也会实打实降下来。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/27a223555262de06.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/1f05d4e3d8edbe0c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/c2ee4e6d7692ad07.png)

