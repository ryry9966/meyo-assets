---
title: Agent 的 tools.md：管理本地配置与环境差异的正确姿势
feedId: 36128
source: 综合讨论
publishedAt: 2026-09-05
---

## 背景

OpenClaw 的 agent 是真的会在你机器上执行命令的：装依赖、翻日志、调你自己的脚本。但它对"这台机器长什么样"没有天生直觉——macOS 笔记本、家里的 Debian 小主机、一台海外 VPS，包管理器、Python 版本、代理、常用路径全都不一样。`tools.md` 就是给 agent 看的本机备忘录：动手前先读一遍，避免瞎猜。

## 问题

不写 tools.md 时，常见的翻车姿势：

- 在 Linux 上跑 `brew`，在 macOS 上试 `apt`；
- 假设项目在 `~/Documents`，实际在 `~/work`；
- 不知道你走代理，pip 拉包超时三次后开始乱试；
- 把这些信息塞进系统提示词或 SOUL.md——那是放人设和长期规则的地方，环境细节堆进去又臭又长，换台机器还全错。

一句话：**环境事实和人设混写，是 agent 配置最常见的腐化来源。**

## 做法与步骤

1. **每台机器一份**，放在 agent 工作区根目录，纳入 dotfiles 仓库统一管理。
2. **结构建议五段**：系统与包管理、常用路径、网络（代理/镜像）、自建脚本、禁区。
3. **只写 agent 不该猜的**：非标准路径、代理、别名、绝对不能碰的目录。它自己 `ls` 一下就能发现的东西不用写。
4. **密钥绝不进 tools.md**，只写引用："用 `OPENCLAW_X_TOKEN` 环境变量"或"走 1Password CLI 取"。
5. **自维护**：在提示词里加一句"发现新的环境事实时，建议追加到 tools.md"，用 git diff 审它提交的内容。

一个 60 行以内的模板：

```markdown
# tools.md — homo-server
## 系统与包管理
- Debian 12；装东西用 apt，别用 snap
- Python 统一走 uv，禁止系统级 pip install
## 路径
- 项目根：~/work/；笔记：~/notes/（syncthing 同步）
## 网络
- 拉 npm/pip 包前先：export https_proxy=http://127.0.0.1:7890
## 自建脚本
- ~/bin/backup.sh：动 ~/notes 下的文件前先跑它
## 禁区
- /etc/nginx/ 有手工调优，未经确认不要改
```

另外注意边界：哪些 MCP server 在哪台机器可用，属于网关配置（`openclaw.json`）的事；tools.md 管"这台机器的物理事实"，两者别互相越界。

## 踩坑点

- **明文写 token**：tools.md 会被完整读进上下文，还可能随日志、截图外泄。密钥只留引用名。
- **写太长**：每次会话都占注意力，超过百行后 agent 开始忽略细节。宁少勿多。
- **过期信息比没有更糟**：换了 fish 却写着 zsh，agent 会理直气壮地用错。改环境当天就更新。
- **多机复制粘贴忘了改主机特有部分**：建议公共部分一份（common.md），每台机器只维护增量。
- **把它当权限系统**：tools.md 是备忘录不是沙箱。真正的限制要靠 OpenClaw 的权限配置来做，备忘录只是"提醒"，挡不住真要乱来的执行。

## 可复用建议

- 把"改环境必改 tools.md"写进个人习惯清单，和改 `.zshrc` 放进同一个 commit。
- 文件头部写一行 last updated，方便自己发现腐烂。
- 定期让 agent"逐条验证 tools.md"，它会自己报出过期条目——这是我用过最省事的清理方式。
- 新机器初始化时，先花十分钟手写一版，再让 agent 补充，比让它从零探索快得多。

## 总结

tools.md 解决的不是"agent 聪不聪明"，而是"它对不对我这台机器的了解"。把它当一份小而准、随环境演进的本机说明书，执行成功率会有肉眼可见的提升；忘了维护，它就是一张会误导人的旧地图。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/0a7d06aa7d3adf50.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/b16e9edad0aff574.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/a366b6b1c64528e9.png)

