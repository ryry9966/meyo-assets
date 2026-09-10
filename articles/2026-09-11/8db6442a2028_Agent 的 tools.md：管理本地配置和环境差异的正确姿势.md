---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 36967
source: 综合讨论
publishedAt: 2026-09-11
---

## 背景

OpenClaw 的 workspace 里有几个约定文件：`AGENTS.md` 管行为规范，`SOUL.md` 管人格设定，而 `tools.md` 的定位更朴素——告诉 Agent 这台机器上的工具事实。它会被加载进上下文，相当于 Agent 的"本地环境说明书"。

不少人做自动化时忽略这个文件，结果就是：Agent 每次执行命令都在猜。

## 问题

典型场景：

- 开发机是 macOS（brew、`/opt/homebrew`），服务器是 Debian（apt、`/usr/bin`），Agent 在服务器上照样敲 `brew install`；
- 本机 Python 走 pyenv，Agent 调用了系统自带的 3.9，装包装到一半发现模块找不到；
- 同一个纠错反复发生：告诉过它 "ffmpeg 在 /usr/local/bin"，三天后它又忘了。

这不是能力问题，是环境信息没有落在 Agent 每次都能读到的地方。对话里说一次会失效，写进 tools.md 才是持久化解法。

## 做法

**1. 建立最小结构。** 在 workspace 根目录维护 tools.md，按"事实条目"组织：

```markdown
# 环境
- OS: 服务器 Debian 12，开发机 macOS 14
- 包管理: 服务器用 apt，禁止使用 brew

# 工具路径
- ffmpeg: /usr/bin/ffmpeg (6.1)
- python: pyenv 管理的 3.12，路径用 `pyenv which python` 获取

# 约定
- 测试服务端口固定 18789
- 临时文件一律写 /tmp/agent-work，禁止写家目录

# 密钥
- 一律从 ~/.agent.env 读取，本文件不出现任何 secret
```

**2. 只写事实，不写愿望。** "尽量用 3.12" 是愿望，Agent 不好执行；"python 路径用 `pyenv which python` 的输出" 是事实。Agent 需要确定性。

**3. 控制篇幅。** tools.md 全量进入上下文，建议 100 行以内。写多了不是没风险，是把风险转嫁给了指令遵循能力。

**4. 机器差异用脚本解决。** 写一个 `gen-toolsmd.sh`，输出当前机器的 which/版本快照，替换"工具路径"一段。多机场景：公共部分进 git，机器相关段落由脚本生成，不手维护。

## 踩坑点

- **把 secret 写进 tools.md。** 它会进 prompt，日志、截图、上下文导出都可能泄露。密钥只放 .env 类文件，这里只写路径。
- **堆积过时信息。** 升级系统后路径变了，Agent 继续用旧路径，排查半天发现是说明书本身错了。换机或升级后第一件事：diff tools.md。
- **和 AGENTS.md 职责混淆。** "不要执行 rm -rf" 是行为规范，放 AGENTS.md；"docker 命令需要 sudo" 是环境事实，放 tools.md。混着写两边都会被稀释。
- **留注释和草稿。** Markdown 里没有 Agent 会忽略的注释，写了"待定"的内容它也会当真。

## 可复用建议

- 每个条目一行、可验证，Agent 能直接执行核对；
- 团队场景：仓库放 tools.md 模板 + 生成脚本，个人差异留在本地覆盖；
- MCP 工具同样适用：哪个 server 在哪台机器可用、哪些工具被禁用，都值得写明；
- 把 tools.md 的 review 纳入换机/升级 checklist。

## 总结

tools.md 的价值不在"多写"，而在"写对"：只写这台机器上可验证的事实，控制长度，密钥外置，版本可控。它解决的是 Agent 自动化里最廉价也最高频的失败——环境猜错。花二十分钟把说明书写好，比事后排查十次"为什么命令不对"划算得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/121349ad0f09f551.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/6aa06b01457e6a15.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/79b2ae1268af721a.png)

