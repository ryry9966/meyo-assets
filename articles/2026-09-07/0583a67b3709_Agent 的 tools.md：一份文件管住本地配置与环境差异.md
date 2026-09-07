---
title: Agent 的 tools.md：一份文件管住本地配置与环境差异
feedId: 36482
source: 综合讨论
publishedAt: 2026-09-07
---

## 背景

跑 OpenClaw 或任何带 MCP/插件的 Agent，最烦的不是写 prompt，而是“这台机器和那台机器不一样”。笔记本上是 Homebrew + Python 3.12，家里的小主机是 apt + 3.10，CI 容器里连 pip 都要走内网源。Agent 每次动手前都在猜：用哪个包管理器、路径在哪、哪个 MCP server 已经起了。猜错的代价是实打实的——装错环境、跑错目录、调不存在的工具。

## 问题

常见的三种错误姿势：

1. **硬编码进 system prompt**：换台机器就得改一遍，还会把本机绝对路径泄进共享配置；
2. **什么都不写，靠 Agent 自查**：每轮会话浪费一串探测命令，而且 `python` 到底指向谁，它未必猜得对；
3. **散落在 README、注释、口头约定里**：三个文档三个版本，谁也说不清哪个是新的。

## 做法

我的方案是分层维护一份 `tools.md`，把它当成 Agent 的运行时清单，而不是文档。

**1. 分两层**

- `~/.openclaw/tools.md`：机器级事实（OS、shell、语言版本、包管理器、代理、GPU）；
- 项目根目录一份项目级：只写工具链和约定，可覆盖机器级的同名条目。

**2. 固定结构**，示例如下：

```markdown
# tools.md (project)
## 环境
- python: 3.12 (pyenv), node: 20.x (nvm)
- 包管理: uv（不要直接用 pip）
## MCP / 插件
- filesystem: 已启用, 根目录限定为本仓库
- playwright: 按需启动, `npx playwright`
## 约定与禁区
- 一律用 `uv run`, 禁止全局安装
- `data/` 只读, 不要写入 `archive/`
```

**3. 半自动生成**：写个十几行的 shell 脚本 dump 版本和路径，输出到标了 `<!-- auto -->` 的区块；人工只维护“约定与禁区”这类机器写不出来的部分。环境变了重跑一次。

**4. 接进 Agent**：在会话引导（bootstrap）里加一句固定指令——“开始任何任务前先读 tools.md，与环境相关的判断以它为准”。这一步不做，文件写了也白写。

## 踩坑点

- **写了但没接线**：最常见的失败是文件躺在仓库里，Agent 从来没读。引导指令必须进 bootstrap，不能靠人肉提醒。
- **过期内容比没有更糟**：版本号是快照不是事实。auto 区块加个更新日期，或者干脆每次会话生成。
- **贪长**：Agent 上下文很贵，只写“会影响它决策”的事实，控制在 150 行以内，否则等于往 prompt 里倒垃圾。
- **敏感信息**：机器级那份含本机路径和内网地址，进 `.gitignore`；提交到仓库的项目级版本先脱敏。
- **md 和脚本别混**：tools.md 是给 Agent 看的，探测和安装逻辑放进可执行脚本，md 里引用脚本名即可。

## 可复用建议

- 把 tools.md 当代码管理：进版本库、走 review，改环境必须同步改它；
- 排查“在我机器上能跑”类问题，先 diff 两台机器的 tools.md，八成差异就在里面；
- 团队共享同一份模板，新人入职 = 填模板 + 跑生成脚本，十分钟对齐环境。

## 总结

tools.md 的价值不在“多一份文档”，而在把环境差异从 Agent 的猜测变成显式输入。分层、半自动生成、接进会话引导、控制长度——做到这四点，它就是一个低成本、高复用的运行时清单。别把它写成散文，当成清单和契约来维护就对了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/6f4d66fb5756958a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/a2bf99818af05e4c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/705918855d70ee1f.png)

