---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 36745
source: 综合讨论
publishedAt: 2026-09-09
---

## 背景

用 OpenClaw 跑自动化久了，都会遇到同一件事：agent 能调 shell、能接 MCP，但它对"你这台机器"一无所知。python 是 `python` 还是 `python3`？用 conda 还是 uv？代理走哪个端口？这些信息要么靠 agent 现场探测，要么靠它在系统提示里碰运气。

借鉴 AGENTS.md 的思路，我们在 workspace 里加了一个 `tools.md`，专门记录本机环境事实，让 agent 开工前先读。

## 问题

没有 tools.md 时的典型症状：

- agent 每个任务开头先跑五条探测命令，烧 token 也烧轮次；
- 更糟的是它直接按训练记忆假设环境——默认有 conda、默认 macOS 路径——报错后进入瞎猜重试循环；
- 多台机器配置漂移，同一个 prompt 在办公本能跑、在家里报错；
- 环境信息硬编码进系统提示，换机器就得改提示词，没法版本管理。

## 做法

1. **建文件，分主题。** 在 workspace 根目录建 `tools.md`，按节组织：运行时与包管理（解释器路径、venv 位置、激活方式）、常用命令（启动/测试/构建，写清"在哪个目录执行"）、网络（代理、镜像源、内网服务地址）、禁区（不要动的目录、不要 kill 的进程、禁止 sudo 的命令）。
2. **区分固定事实和易变状态。** 路径、别名写死；进程列表、磁盘余量这类易变信息只写探测方法，不写具体值。
3. **约定探测协议。** 遇到文档没覆盖的环境，明确先跑哪几条命令，把结论回写 tools.md 对应小节，改完汇报，人来 review。
4. **多机用 per-host 拆分。** tools.md 放公共部分，本机差异拆到 `tools.<hostname>.md`，主文件里引用。
5. **进 git。** 环境变更和环境文档变更同一个 commit 提交。

## 踩坑点

- **写成教程而不是事实清单。** tools.md 是"这台机器的现状"，agent 会原文引用，教程式内容白白吃掉上下文。
- **明文写密钥。** tools.md 会被注入上下文、可能进日志。密钥一律走环境变量，文件里只写"从 `~/.xxx` 读取"。
- **信息过期比缺失更糟。** agent 默认信任文档，改了环境没回写，排障方向直接被带偏。建议配一个 env-probe 脚本，定期 diff 现实与文档。
- **篇幅失控。** 超过两三百行就该拆，长细节放子文档让 agent 按需读。
- **平台差异混写。** macOS 和 Linux 的命令不分节，agent 会挑错分支。

## 可复用建议

- tools.md + env-probe 脚本配套：脚本输出事实，agent 对照回写，人只看 diff。
- 在 bootstrap 提示里固定加一句："开工前先读 tools.md；文档与实际冲突时，以探测结果为准并回写。"
- MCP 服务器清单、本地端口约定也纳入进去，能省掉一大类端口冲突排障。
- 公共部分入 git，隐私段落进 `.gitignore` 或本地文件，别为了同步牺牲安全。

## 总结

tools.md 的本质，是把环境假设从 prompt 里的隐性知识，变成显性、可 review、可版本化的资产。它不替代探测，而是让 agent 的第一步从"猜"变成"查"。落地成本很低：今天先把"python 在哪、怎么跑测试、代理怎么配"三行写进去，跑一周再迭代。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/f046bfdb961cbe35.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/7f1dd38fc4dece4f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/a301467fdb60158c.png)

