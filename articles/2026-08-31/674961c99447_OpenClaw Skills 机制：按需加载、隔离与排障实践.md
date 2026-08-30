---
title: OpenClaw Skills 机制：按需加载、隔离与排障实践
feedId: 35458
source: 综合讨论
publishedAt: 2026-08-31
---

## 背景

在 OpenClaw 的 Agent/自动化场景里，不少人的第一反应是挂插件或接 MCP。插件和 MCP 工具通常会在会话启动时注册，甚至常驻运行。能力少的时候没问题，但当你同时给助手装了十来个工具，问题就开始出现：上下文窗口被工具描述挤占、工具选择路由不稳、不同插件之间的依赖冲突、权限暴露面过大。

OpenClaw Skills 提供的是另一种思路：默认只保留一个轻量级能力索引，真正要执行某个技能时，才按需把代码、提示词、依赖和资源挂载进来。加载、执行、卸载有明确生命周期。

## 问题

一个常见例子是：工程师给助手装了一个 PDF 处理、一个表格分析、一个浏览器自动化，再加几个 MCP。每次新会话都全量加载，Token 成本高不说，两个工具都声明了“读取文件”或“搜索网页”，Agent 经常选错。

Skills 机制更接近于“按需加载的执行单元”，而不是“常驻插件”。默认进入索引的是 manifest，不是完整实现。命中触发条件后才加载，执行结束按策略卸载。这能省 token、降低工具选择噪声，也让权限边界更清楚。

## 做法/步骤

### 1. 用 manifest 描述能力边界

一个 skill 目录里放 manifest 和入口。重点是 description 和 triggers 要克制：描述“什么时候不该用”比“能做什么”更有用。

```yaml
# skills/pdf-summarizer/skill.yaml
name: pdf-summarizer
version: 0.1.0
description: Extract text from PDFs and produce summary. Use only when user mentions PDF or document.
triggers:
  keywords: ["pdf", "文档", "摘要"]
  intents: ["document_analysis"]
  requires_attachment: true
entrypoint: ./run.sh
permissions:
  files: ["read:attachment"]
  network: false
resources:
  max_memory_mb: 512
lifecycle:
  setup: ./setup.sh
  teardown: ./teardown.sh
```

### 2. 注册仓库并开启 lazy load

```bash
openclaw skills register ./skills --enable-lazy --priority 50
openclaw skills list --status
```

注册后只会构建索引，不会执行 setup。状态里能看到哪些是 indexed、哪些是 loaded。

### 3. 触发按需加载并观察 trace

```bash
openclaw run "帮我把这份 PDF 做摘要" --trace skill --session-id test-01
```

关键 trace 点依次是：resolve -> load -> setup -> run -> teardown。如果 resolve 阶段没命中，实现代码不会被加载。

### 4. 设置卸载与预算

```yaml
runtime:
  max_loaded_skills: 3
  eviction: lru
  load_cooldown_seconds: 30
```

这能避免多个技能在同一会话里反复挂载、卸载，造成资源抖动。

## 踩坑点

1. **触发词太宽或太窄**。用户说“读一下这份合同”，结果没命中“PDF”技能；或者用户只是说“文件”，就误加载了 PDF、表格、OCR 三个技能。建议先跑 shadow mode，看候选命中率再调整。
2. **teardown 不干净**。技能里起了本地端口或子进程，teardown 只杀了主进程，端口还被占用。需要写 PID 文件或用 cgroup/容器隔离。
3. **manifest 权限声明不诚实**。manifest 写 `network: false`，run.sh 里却偷偷 curl 外部 API。按需加载不等于自动安全。
4. **多个技能竞争同一 trigger**。用户说“分析数据”，表格技能和 Python 技能都被命中，来回切换。需要优先级、冷却时间，必要时让 Agent 先向用户澄清。

## 可复用建议

- **把小技能作为默认单元**：一个 skill 只做一类事情，避免巨型技能重新变成常驻插件的翻版。
- **把 MCP 工具作为依赖声明**，不要在 skill 内部隐式调用全局 MCP 工具。
- **上线前跑 dry-run/shadow mode**：只看 would-load 日志，不实际执行，用来校准触发质量。
- **每个 skill 独立目录和依赖锁文件**，防止跨技能依赖冲突。
- **排障先看 resolve 阶段**：如果 resolve 没进候选，别急着查 run.sh。

## 总结

OpenClaw Skills 不是另一种插件格式，而是把“能力注册”和“能力执行”解耦。默认只索引轻量 manifest，按任务意图按需挂载，按预算卸载。省 token、降冲突、压缩权限暴露面。工程落地的关键是：manifest 写得克制，生命周期写完整，用 shadow mode 验证触发质量。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/2020462989259a31.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/dbd4581d7bb09d79.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/eee5b510b03abce0.png)

