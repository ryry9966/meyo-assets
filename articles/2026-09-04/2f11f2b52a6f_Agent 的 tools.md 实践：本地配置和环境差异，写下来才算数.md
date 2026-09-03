---
title: Agent 的 tools.md 实践：本地配置和环境差异，写下来才算数
feedId: 35977
source: 综合讨论
publishedAt: 2026-09-04
---

## 背景

跑 Agent 做自动化的人多半遇到过这种场景：同一个任务，在开发机上顺利跑完，换到家里的服务器就报 `command not found`；或者 Agent 反复执行 `which`、`ls` 去试探环境，白烧上下文。行为层面我们有 AGENTS.md 做约定，但"这台机器上到底有什么工具、怎么调"往往没人写下来，全靠模型猜。

tools.md 就是补这块的：一份放在固定位置、随会话注入模型上下文的工具清单与环境说明。成本极低，收益直接。

## 问题拆开看

实际踩到的坑有三类：

1. **存在性差异**：ffmpeg、jq、rg 这类工具 A 机有、B 机没有，Agent 按 A 机经验直接调用，失败后开始瞎猜替代方案。
2. **路径与版本差异**：macOS 在 `/opt/homebrew/bin`，Linux 在 `/usr/bin`；python3 指向 3.9 还是 3.12，行为完全不同。
3. **隐性约定**：某脚本必须先激活 venv，某 API 依赖本地代理——这些只存在于个人笔记里，换个人换台机器就断。

## 做法

核心原则一句话：**tools.md 是 Agent 和环境之间的契约，写清"有什么、怎么调、什么条件"。**

步骤如下：

1. **在仓库根目录建文件**（或用 `~/.openclaw/tools.md` 做全局兜底），每个工具一个条目，字段固定：

```markdown
## ffmpeg
- 用途: 转码、抽帧、提取音轨
- 调用: ffmpeg（已在 PATH）
- 前置: 无
- 限制: 本机无 GPU 编码，大文件加 -preset veryfast
```

2. **区分"保证可用"和"尽力而为"**。前者是脚本可无条件依赖的，后者要标注失败时的降级方案，Agent 对两类的处理策略应当不同。

3. **环境差异用变量表达，不写死路径**。条目里写 `${VIDEO_DIR:-~/Videos}`，单独列一节"必需环境变量"，机器特定的值放 `.env`，不进 tools.md。

4. **配一个 doctor 脚本**。遍历 tools.md 声明的工具，逐个 `command -v` 加版本检查，输出"声明 vs 实际"的差异。换新机器跑一遍，缺什么补什么，十分钟收敛。

5. **控制长度**。这份文件每次会话都进上下文，压在 150~200 行内。细节放独立文档，tools.md 只留索引和关键约束。

## 踩坑点

- **把机密写进去**。tools.md 会整份进模型上下文，API key、内网地址一律不放，只写"需要 OPENCLAW_PROXY，见 .env"。
- **写成散文**。大段描述模型抓不住重点，坚持结构化条目，字段名固定。
- **删了工具忘更新**。Agent 会持续尝试调用已卸载的工具然后反复失败。doctor 脚本要能检出"声明了但不存在"的条目。
- **假设 Agent 一定读到**。确认加载链路真的把 tools.md 注入了系统提示，用一次简单提问验证，比如让 Agent 列出它已知的前三个工具。

## 可复用建议

- 团队共享一份 `tools.md`，个人差异放 `tools.local.md`，加载时后者覆盖前者，避免各自 fork 出分叉。
- 把 doctor 脚本挂进 CI 或启动钩子，让配置漂移在运行前暴露，而不是任务跑到一半才炸。
- 把 tools.md 当代码对待：改条目走 PR，重大变更在文件头写一行变更记录。

## 总结

tools.md 解决的不是技术难题，而是一个常被忽略的工程习惯：Agent 的行为可以靠提示词约束，但它对环境的认知必须有人负责提供。十几分钟写一份结构化清单，换来的是更少的试探性调用和更稳的跨机器复现。配置即契约，写在 Agent 读得到的地方，才算数。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/668ddeba11e1fdad.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/2a4b682aeeff202f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/c17a3d4ea3886764.png)

