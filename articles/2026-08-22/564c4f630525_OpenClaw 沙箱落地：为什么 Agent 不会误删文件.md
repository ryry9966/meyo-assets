---
title: OpenClaw 沙箱落地：为什么 Agent 不会误删文件
feedId: 34191
source: 综合讨论
publishedAt: 2026-08-22
---

## 背景

最近在 OpenClaw 里跑文件整理和自动化任务，最担心的不是 Agent 不做，而是做得太“彻底”。让它清理重复文件，它可能把原始稿一起归档；让它整理下载目录，它可能把 `.ssh` 当垃圾。传统 prompt 约束在工具链复杂后很容易失效，因为 Agent 面对的是真实 shell、文件系统 MCP 和插件，而不是一个被“嘱咐过”的对话模型。

## 问题

OpenClaw 的 Agent 通常不会主动乱删文件，但一个误判的工具调用就能造成不可逆损失。风险主要来自三类：

1. 文件系统边界过大，Agent 可以读写用户目录甚至系统目录；
2. 命令允许列表过宽，裸 `rm`、`shred`、`dd` 等命令可直接执行；
3. MCP/插件默认继承较大权限，比如文件系统 MCP 默认根目录是 home。

大多数误删不是模型突然变坏，而是执行环境给了它越界能力。

## 做法/步骤

### 1. 先收窄文件系统边界

在 OpenClaw 的 sandbox 配置里，只暴露必要目录。示例配置做了简化，具体字段以当前 OpenClaw 版本为准：

```yaml
sandbox:
  fs:
    allowed_roots:
      - /srv/agent/workspace
    writable_roots:
      - /srv/agent/workspace/tmp
      - /srv/agent/workspace/output
    read_only_roots:
      - /srv/agent/workspace/data
```

正式资料用 `read_only`，输出只到 `output`。即使 Agent 想删原始数据，它在文件系统层也没有写权限。

### 2. 禁止裸 rm，只走 trash

删除操作不是完全不给，而是包装。系统里配一个 `trash` 命令，内部用 `gio trash` 或 Python 的 `send2trash`，并加 `--dry-run`：

```bash
#!/usr/bin/env bash
set -euo pipefail
if [[ "${1:-}" == "--dry-run" ]]; then
  shift
  echo "would trash: $*"
else
  gio trash "$@"
fi
```

OpenClaw 命令 allowlist 只放 `trash`，deny `rm`、`shred`、`dd`。这样即使 prompt 被绕过，Agent 也调不到裸删除。

### 3. MCP 默认拒绝

文件系统类 MCP 很危险，因为它把“读取整个目录树”变成一个工具。配置 MCP 时按 server 设置 `default: deny`，只开放必需的 tool/resource。对 filesystem MCP 的 root 限制在 workspace，且写路径与 OpenClaw sandbox 保持一致。

### 4. 写操作和读操作分权

不要让 Agent 同时拥有“读全盘 + 写全盘”。读可以放宽到资料库，但写只允许 `tmp/output`。需要落到正式目录时，走人工确认或脚本队列。

### 5. 保留操作日志

至少记录 tool name、参数、路径、结果。日志不用复杂，JSONL 即可：

```json
{"ts":"2025-01-01T10:00:00Z","tool":"trash","args":["/srv/agent/workspace/tmp/a.txt"],"result":"ok"}
```

出问题时先看日志，而不是先怀疑模型。

## 踩坑点

- **只 allowlist 命令，忘了限制目录**：Agent 用 `trash /home/user/important.txt` 时，命令合法，但文件系统边界失守。目录白名单和命令白名单必须同时生效。
- **把 `.env` 或配置目录暴露在 read_only_roots**：Agent 读取 secrets 后，可能通过其他工具外传。配置目录不要放进 workspace。
- **用 prompt 代替权限**：写“不要删除”是提醒，不是控制。命令注入、多轮误解都能让这句话失效。
- **MCP 默认路径是 home**：很多文件系统 MCP 默认根目录是用户目录，不显式改 root 等于没沙箱。
- **以为 trash 无限保险**：trash 也会占磁盘，需要设置清理周期。

## 可复用建议

- 最小权限三目录：`workspace/tmp` 可写、`workspace/output` 可写、`workspace/data` 只读。
- 删除、覆盖、移动都封装成脚本，并加入 `--dry-run`。
- MCP/插件安装后先做一次越权测试：让 Agent 读 `/etc/passwd`、删除 `workspace/data` 下测试文件，看日志是否被拦截。
- 沙箱配置放 git，变更可追溯。
- 所有清理任务先跑 `--dry-run`，人工确认后再执行。

## 总结

OpenClaw 的 sandbox 安全模型不是让 Agent 更聪明，而是让环境更窄。默认拒绝、最小授权、删除走回收、操作可审计。做到这些，Agent 不会误删文件——更准确地说，它没有能力误删原始文件。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/75ea98bceedf748f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/7f8259febafa2eb5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/b255c889c8cb4ae9.png)

