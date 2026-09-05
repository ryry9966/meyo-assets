---
title: Agent 的 TOOLS.md：管理本地配置和环境差异的正确姿势
feedId: 36205
source: 综合讨论
publishedAt: 2026-09-05
---

## 背景

OpenClaw 的 agent 每次会话都从工作区的几个 Markdown 文件建立认知：AGENTS.md 管行为约定，MEMORY.md 管记忆，而 TOOLS.md 的定位是"当前这台机器的工具与环境事实"。定位很清楚，但实践里它常被写成三种东西之一：空文件、安装教程、什么都塞的大杂烩。结果是 agent 每次都现场摸环境——`which python3`、ffmpeg 在不在、docker 起没起——一轮探测既烧 token 又拖慢首个任务，摸错了还会拿着错误假设一路跑到底。

## 问题

环境差异是常态，不是异常：

- 开发机是 macOS（brew 的 ffmpeg 在 `/opt/homebrew/bin`），家里服务器是 Ubuntu（路径完全不同），VPS 上根本没装；
- 同一件事在不同机器上是 `python`、`python3` 或某个 venv 里的解释器；
- 代理、镜像源、WSL 路径映射、GPU 驱动这些"隐形前提"，agent 默认一概不知。

这些不该靠每次现场推理解决——它们是事实，应当被记录。

## 做法

1. **先分层，再动笔。** AGENTS.md 只放跨机器稳定的约定；TOOLS.md 只放当前机器的事实；密钥、token、内网地址走环境变量和配置文件，一个字不进 Markdown。
2. **按五个板块写，控制在 40 行以内：** 系统概况、工具清单（路径 + 版本 + 验证命令，MCP server 和插件"装没装、怎么启"也一并登记）、网络环境、已知坑、禁区。示例：

```markdown
# 工具
- python3: miniconda，用 `which python3` 确认
- ffmpeg: /usr/bin，4.4

# 网络
- pip 需走代理 http://127.0.0.1:7890

# 已知坑
- apt 里没有 certbot，用 snap

# 禁区
- 不要动 node
```

3. **写"查法"优于写"值"。** 与其写死路径，不如给 agent 一条低成本自证的命令——值会过期，查法不会。
4. **多机方案：一机一文件。** 我试过在一份 TOOLS.md 里用 if/else 描述三台机器，agent 读起来混乱、容易张冠李戴。现在每台机器独立工作区，共享约定放 AGENTS.md，TOOLS.md 直接进 gitignore。
5. **让它活起来。** 在 AGENTS.md 里加一条规则：当命令失败的原因与 TOOLS.md 记录冲突时，先修正文档再继续任务。另备一个 10 行的探测脚本（收集 uname、which、版本号）作为新机器的初始化草稿，人工裁剪后入库。

## 踩坑点

- **写成教程。** agent 不需要"ffmpeg 是什么"，只需要"有没有、在哪、版本多少"。文档越长，关键事实越被稀释。
- **硬编码易变项。** 写死的容器名、临时 IP，两周后就是毒药。易变项一律改成查询方式。
- **秘密进文档再同步。** TOOLS.md 里出现任何 key 都应当按泄漏处理。
- **没有更新机制。** 缺了"冲突即更新"这条，一个月它就腐烂成误导源。

## 可复用建议

二十字口诀：**一机一文件、事实非教程、查法优于值、秘密不进文档、冲突即更新。** 五条全做到，TOOLS.md 是资产；缺一条，它迟早变负债。

## 总结

TOOLS.md 的价值不在于"写了文档"，而在于把 agent 对环境的认知，从每次会话的现场推理，变成可读、可验证、可维护的持久状态。它只有几十行，省下的是每次会话的探测成本和每次错误假设的排查成本。环境差异消灭不了，正确姿势是让差异被准确记录、低成本验证。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/81b013ca85fcac55.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/a06c563ef28142e3.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/4cf66d9af6bd9dbe.png)

