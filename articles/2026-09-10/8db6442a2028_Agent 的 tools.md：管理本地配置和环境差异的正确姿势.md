---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 36900
source: 综合讨论
publishedAt: 2026-09-10
---

## 背景

Agent 跑起来之后，真正反复消耗你时间的往往不是任务本身，而是它每次都要重新认识你的机器：这个环境用 brew 还是 apt？是 python 还是 python3？某个 MCP server 到底注册了没有？这些信息散落在 shell rc、.env、MCP 配置文件和你的脑子里，agent 只能靠现场试探，猜错一次就是一轮重试和烧上下文。

tools.md 的定位很朴素：把"这台机器/这个项目有什么工具、有什么约定"固化成一份受版本管理的清单，放在 agent 一定会读的地方。

## 问题

没有这份清单时，常见症状包括：

- 同一任务在 macOS 笔记本、Linux CI、容器里表现不一致。典型如 BSD sed 和 GNU sed 的 `-i` 参数差异，agent 在 macOS 上跑 GNU 写法直接报错。
- MCP 注册信息和实际环境脱节：文档里写了三个 server，实际只起了两个，agent 反复调用失败的那个。
- 多人（或多 agent）共用仓库，各自踩一遍环境坑，知识无法沉淀。
- 环境事实和密钥混着写，一不小心 API key 就进了上下文和日志。

## 做法

核心原则：**tools.md 只记录事实，不承担操作职责**。推荐分区：

```markdown
# tools.md
## platform
- macOS 14 / arm64，Homebrew；GNU 工具用 g 前缀（gsed、gawk）
## runtimes
- node 22（fnm 管理），自检：node -v
- python 3.12（uv 管理），自检：uv --version
## mcp
- filesystem: npx xxx-server-filesystem ./（stdio，项目级）
- sqlite: uvx mcp-server-sqlite --db-path ./data.db
## env
- OPENCLAW_API_KEY：必填，值见 .env，本文件只写名字和用途
## quirks
- sed -i 一律用 gsed；路径避免绝对用户目录
```

落地步骤：

1. **分层**。全局环境放 `~/.openclaw/tools.md`（个人机器事实），项目根放项目依赖和 MCP 注册，主机差异用 `tools.darwin.md` / `tools.linux.md` 小文件叠加，避免单文件里 if-else 堆积。
2. **用探测脚本生成初稿**。用 `command -v`、`--version` 扫一遍常用工具，吐出草稿，人工审核后入库。生成物只是起点，不是自动维护机制。
3. **约定 agent 行为**：执行前先读 tools.md；发现文档与现实不符时，报告差异，而不是擅自改文档或绕过。
4. **和现实锁死**。把清单里的自检命令抽出来在 CI 跑一遍，文档过期就红。

## 踩坑点

- **把密钥写进 tools.md**。这个文件会整体进入上下文，等于公开。
- **越写越长**。超过一百行就该删减或拆分，上下文不是免费的。
- **写绝对路径**。`/Users/zhang/...` 换台机器即废，统一用相对路径加环境变量。
- **写成教程**。"如何安装 node"属于 README；tools.md 只回答"有没有、什么版本、怎么验证"。
- **放任 agent 自动追加**。每次运行都把新发现的工具写回去，几轮之后文件自相矛盾。变更必须过人。

## 可复用建议

- 每条记录尽量带一行自检命令，agent 先验证再使用，减少"想当然"。
- 定期让 agent 做环境审计：输出它理解的工具清单，和你自己的认知 diff 一次，漂移早暴露。
- 环境事实（tools.md）和环境变更（安装脚本）分离，文档不背执行的锅。
- 团队场景下把 tools.md 纳入 code review：环境约定和代码约定同等对待。

## 总结

tools.md 本质上是把环境知识从口口相传变成单一事实源。它不解决"装环境"的问题，只解决"认识环境"的问题——让 agent 少猜、少错、少烧上下文。维护成本很低，收益随机器和协作者数量线性增长，属于典型的低成本高杠杆动作，建议每个跑 agent 的仓库都配一份。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/093161cee05a682c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/236210f7a0f41555.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/8403c7c5c48287de.png)

