---
title: Agent 的 tools.md：管理本地配置与环境差异的工程化做法
feedId: 36591
source: 综合讨论
publishedAt: 2026-09-08
---

Agent 干活的最后一步几乎都在本机执行：跑脚本、调 ffmpeg、处理数据、git 提交。模型对“标准 Linux”很熟，但你的机器从来不是标准的。这类差异模型猜不到，猜错就是一次失败的工具调用。本文讲怎么用 workspace 里的 tools.md 把这件事管起来。

## 背景：环境差异是怎么坑 Agent 的

- macOS 上是 `python3`，某些容器里只有 `python`；
- brew 装的 ffmpeg 在 `/opt/homebrew/bin/`，apt 装的在 `/usr/bin/`；
- 家里服务器有 docker，笔记本没有；出网要走代理，或者根本不通。

这些是“这台机器特有的事实”，每次会话都重讲一遍不现实，全靠模型记忆也不可靠。

## 问题：三种常见的错误姿势

1. **写死在 system prompt**——换台机器全失效，还长期占用上下文；
2. **塞进长期记忆 / MEMORY.md**——环境事实更新频繁、和机器绑定，混在用户偏好里很快腐烂；
3. **让 agent 每轮自己探测**——能跑，但 `which`、`--version` 反复执行又慢又烧 token，结果还容易在多轮间丢失。

结论：环境事实需要一个按机器维护、agent 一定会读的独立位置。这就是 tools.md 的定位。OpenClaw 的 workspace 里天然有它的位置（`~/.openclaw/workspace/tools.md`，不同版本命名略有差异）；如果你的框架没有内置约定，就在 AGENTS.md 或系统提示里加一条：“使用不熟悉的工具前，先读 tools.md”。

## 做法

### 1. 先划边界：只写“探测不出来”的事实

模型自己能 `which` 到的不用写。值得写的是：非默认路径和版本差异、调用约定（必带 flag、习惯参数）、网络与权限约束、你希望它遵守的策略。

### 2. 一个能用的模板

```markdown
# tools.md — 本机工具环境（2025-06-12 校验）

## python
- 命令：python3（没有裸 python）
- 包安装必须走 ~/venvs/agent，系统 python 装不了包

## ffmpeg
- 路径：/opt/homebrew/bin/ffmpeg（brew）
- 约定：转码统一 -hwaccel videotoolbox

## 网络
- 出网走 http://127.0.0.1:7890；docker 内走 host.docker.internal:7890

## 明确没有
- docker、npm 全局安装（别尝试，改用 python 方案）
```

“明确没有”这段最容易被忽略，但性价比最高——它直接掐断了 agent 的错误尝试方向。

### 3. 用脚本生成事实区，不要手抄

环境事实是机器可探测的。写一个 `tools-doctor.sh`：把关键二进制的路径、版本、代理变量、目录存在性检查一遍，输出合并进 tools.md 的**事实区**；你的策略放在**约定区**，只手工维护这一块。新机器装机或大版本升级后跑一次。

关键是心态：tools.md 是可再生成的缓存，不是需要精心供奉的文档。这个定位决定了它不会腐烂。

### 4. 多机与密钥

- tools.md 跟机器走，不进 dotfiles 仓库；仓库里只放 `tools.template.md`（约定区），新机器跑 doctor 填充事实区；
- 密钥绝不进 tools.md，走环境变量，md 里最多写“KEY 从 env 读取”。

## 踩坑点

- **写成人类文档**：三页纸的 tools.md 每轮进上下文，token 和注意力一起遭殃，控制在几十行；
- **信息过期**：brew 升级后行为变了。头部加校验日期，doctor 脚本顺带刷新，别指望手工维护；
- **agent 不一定读**：光有文件没用，AGENTS.md 里必须有触发规则——“执行外部命令前查 tools.md，冲突时以 tools.md 为准”；
- **把偏好当事实**：任务上下文归 memory，tools.md 只放环境事实和约定；
- **多机共用一份**：两个环境共用 tools.md，等于没有 tools.md。

## 可复用建议

1. 一行判断标准：“这条信息 agent 能否在一秒内自己探测到？”能就不写；
2. “事实区自动生成 + 约定区手工维护”的分层结构，适用于几乎所有配置管理场景；
3. “明确没有 / 别做”清单优先写，收益立竿见影；
4. 校验自动化：doctor 脚本挂开机钩子或 cron，过期即重算。

## 总结

tools.md 解决的不是“工具怎么用”，而是“这台机器上工具长什么样”。把它当成机器事实的可再生缓存，配一个生成脚本和一条读取规则，agent 在笔记本、家庭服务器、云容器之间切换时，就会少很多“上次明明能跑”的玄学时刻。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/06f790f49e76b3b2.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/155ff83775eca09f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/c9250c90b7bcdbad.png)

