---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 37102
source: 综合讨论
publishedAt: 2026-09-12
---

## 背景

OpenClaw 这类 Agent 框架依赖 markdown 注入运行上下文，`tools.md` 的职责是告诉 Agent：这台机器上有什么工具、怎么调、什么时候不能用。单人单机时随手写几行就能跑，但只要换一台电脑、或多个人协作同一份仓库，问题立刻暴露。

## 问题

典型故障有三种：

1. **路径写死**。`/usr/local/bin/ffmpeg` 在你的 mac 上存在，在同事的 Ubuntu 上是 `/usr/bin`，在容器里干脆没有。
2. **环境不继承**。Agent 的执行环境往往不是你的交互 shell——由 systemd/launchd 拉起的进程拿不到 `.zshrc` 里的 PATH、代理和 conda 激活，连 `python` 和 `python3` 都可能找不到。
3. **多处真相**。`tools.md`、MCP 配置、dotfiles 各写一份，改了 MCP 忘了 md，Agent 按过期信息调用，然后开始"现场发挥"。

## 做法：拆成三层

核心思路是把"约定"和"实例"分开：

```text
tools.md              # 入库：能力声明 + 调用契约，不含路径和秘密
tools.local.md        # gitignore：本机实例，路径/版本/代理/GPU 状态
scripts/preflight.sh  # 探测环境，生成并校验 local 文件
```

`tools.md` 里每个工具固定四个字段：

```text
- name: ffmpeg
  check: ffmpeg -version
  invoke: ffmpeg [args]
  fallback: 不可用则报错退出，禁止用替代品拼凑
```

`tools.local.md` 由脚本生成：探测到 `/opt/homebrew/bin/ffmpeg`，就把绝对路径写进去。`preflight.sh` 在会话启动前跑一遍，逐个执行 `check`，任何一项失败直接阻止启动——而不是让 Agent 跑到一半才发现。

Agent 侧约定两条：会话开始只读这两份文件；任何工具调用前先执行 `check`，禁止假设工具存在。

## 踩坑点

- **别写模糊描述**。"ffmpeg 大概在 /usr/local 下面"这种话 Agent 会当真，然后真的去那里找。要么精确，要么删掉。
- **忘了 gitignore**。`tools.local.md` 里常有 API key 和内网地址，加一条 pre-commit 检查兜底。
- **清单膨胀**。写了五十个工具，Agent 就会尝试五十个。只列当前工作流真正用到的，其余归档。
- **与 MCP 双写漂移**。如果工具同时由 MCP server 提供，让生成脚本以 MCP 配置为单一来源产出 `tools.md` 对应段落，不要手工维护两份。

## 可复用建议

- 把 `preflight` 挂到 session start hook，做成"启动即校验"，比文档里写一句"请先检查环境"可靠得多。
- 每个工具都写 `fallback`，明确不可用时的行为。压缩 Agent 自由发挥的空间，就是压缩事故面。
- `tools.md` 的变更走 code review——它和接口定义同级，不该是某个人的本地笔记。

## 总结

`tools.md` 不是文档，是 Agent 与机器之间的接口契约。环境差异永远存在，正确的姿势不是消灭它，而是把它压缩到唯一一层（local 文件 + 生成脚本），让入库的部分保持通用、可校验、可 review。做到这一点，换机器的成本就从"排障半天"变成"跑一次脚本"。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/883f015384c578c3.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/d6373c66ea61a693.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/b41333811ef7cdc4.png)

