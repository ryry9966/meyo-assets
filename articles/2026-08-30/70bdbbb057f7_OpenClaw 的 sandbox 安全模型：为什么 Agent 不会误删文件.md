---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 35381
source: 综合讨论
publishedAt: 2026-08-30
---

# OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件

## 背景

在 OpenClaw 上跑 Agent、MCP 工具或自动化插件时，文件操作是最常见的动作：读配置、写日志、移动产物、清理临时目录。但一旦 Agent 拿到“删除文件”的能力，误删风险就会成倍放大。一个模糊的 prompt、一次路径拼接错误、或者插件返回了意料之外的路径，都可能让 `rm -rf` 指向错误的位置。

传统做法是给 Agent 当前用户权限，依赖“提示词约束”来避免危险操作。工程实践反复证明，这不够可靠：模型会误解指令，工具链可能有 bug，恶意输入也可能诱导删除。因此 OpenClaw 的 sandbox 安全模型把文件系统访问从“默认允许”改成“默认拒绝、显式授权”，从机制层减少误删概率。

## 问题

最容易出事的场景不是 Agent 主动作恶，而是“越界操作”。例如：

- Agent 被要求“清理当前目录下所有临时文件”，但当前目录被解析成 `~/` 而不是 `~/workspace/tmp`。
- 插件返回的路径包含 `../`，导致删除穿透到上一级。
- 符号链接把沙箱内的路径指向了 `/etc` 或 `~/.ssh`。
- 某个 MCP 工具把“删除”实现为直接调用宿主的 `rm`，绕过了应用层检查。

如果只有 prompt 层面的安全提示，这些情况基本防不住。Sandbox 层的价值在于：无论 Agent 想做什么，文件系统操作都必须经过策略引擎；不在白名单内的路径，连“删除”这个系统调用都发不出去。

## 做法与步骤

下面是一个可复现的最小配置示例。假设你只希望 Agent 在一个工作目录内可写，其他位置全部只读或完全不可见。

1. **初始化沙箱根目录**

```bash
openclaw sandbox init --root ./workspace
```

root 成为 Agent 的文件系统边界，默认情况下 Agent 无法访问 root 之外的任何路径。

2. **显式挂载只读数据目录**

```yaml
sandbox:
  root: ./workspace
  mounts:
    - host: ~/data
      path: /data
      mode: ro          # read-only
    - host: ~/output
      path: /output
      mode: rw          # 仅此目录可写
  deny_paths:
    - ~/.ssh
    - ~/.aws
    - /etc
  symlink_policy: reject # 拒绝跟随符号链接
```

这个配置里，Agent 能写的位置只有沙箱内的 `/output`，而 `/data` 是只读的。即使 Agent 执行 `rm -rf /data`，沙箱也会拒绝。

3. **运行任务并观察拦截日志**

```bash
openclaw agent run --task "清理输出目录" --sandbox-log /tmp/sandbox.log
```

检查日志中 `DENY` 或 `PERM_DENIED` 记录，确认哪些访问被拦截。

4. **验证误删场景**

故意让 Agent 尝试删除 root 外的文件，例如：

```text
请删除 ~/.bashrc
```

在沙箱开启时，该操作应被拒绝，并返回类似“permission denied: path outside sandbox root”的错误。不要在生产机直接测试，建议在容器或虚拟机里做。

5. **开启 dry-run 模式**

对于删除、重命名、覆盖写等高风险操作，先跑一次 dry-run：

```bash
openclaw agent run --task "清理输出目录" --dry-run
```

让 Agent 输出计划中的删除列表，人工确认后再执行。

## 踩坑点

实际落地时，有四个容易忽略的问题：

**符号链接逃逸。** 如果沙箱内存在指向外部的符号链接，而策略允许跟随符号链接，删除操作可能穿透到宿主机。务必设置 `symlink_policy: reject` 或 `no-follow`。

**插件绕过沙箱。** 沙箱通常限制的是 OpenClaw 内置文件工具，但如果加载了第三方插件，插件可能直接调用宿主机的 shell。需要审查插件权限，或者使用进程级沙箱（如 bubblewrap、firejail、Docker）作为第二层防护。

**过度授权。** 把整个家目录以 `rw` 挂载给 Agent，相当于没有沙箱。最小权限原则是：只给任务必需的可写路径；只读路径能不给就不给。

**“删除”不等于真删。** 有些沙箱会把删除操作重定向到回收站或生成快照，看似安全，但长期运行会积累大量垃圾文件，占满磁盘。建议定期清理沙箱快照目录，并设置容量上限。

## 可复用建议

- **默认只读，显式挂载可写目录。** 不要图省事把大目录直接 `rw`。
- **关闭符号链接跟随。** 这是防路径穿越最有效的一行配置。
- **对删除操作做二次确认或白名单。** 例如只允许删除 `/output/tmp/**` 模式匹配的文件。
- **开启审计日志。** 记录所有删除、重命名、写操作，便于事后追溯。
- **定期做恢复演练。** 验证快照或回收站能否真的恢复被“删除”的文件，别等到真出事再测试。
- **多层防护。** OpenClaw 沙箱是文件系统层，配合容器/虚拟机隔离会更稳。

## 总结

OpenClaw 的 sandbox 安全模型不能保证 100% 不误删，但它把“误删”从概率事件变成了需要多层绕过的确定性拦截。核心是默认拒绝、显式授权、路径边界和操作审计。对跑 Agent 自动化的人来说，这套机制比反复在 prompt 里写“不要删除重要文件”可靠得多。配置时保持最小权限，踩坑点提前规避，基本就能避免大多数文件事故。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/d0256e13ff9fa3af.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/c06e0364111a8d48.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/cc8360cc267127a8.png)

