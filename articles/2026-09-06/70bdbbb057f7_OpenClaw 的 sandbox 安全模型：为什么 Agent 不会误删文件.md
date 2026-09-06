---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 36312
source: 综合讨论
publishedAt: 2026-09-06
---

## 背景

OpenClaw 的 Agent 可以执行 shell 命令、读写文件、调用 MCP 工具。能力越大，风险越直接：一次错误的 `rm -rf` 就可能清空工作目录。社区里最常被问的问题之一就是——“让 Agent 自己跑命令，真的安全吗？”这篇帖拆解 OpenClaw 的 sandbox 模型，说明误删文件这类事故为什么在正确配置下很难发生。

## 问题本质

“模型不可靠”是前提，不是 bug。任何依赖“Agent 永远不犯蠢”的安全设计都是脆弱的。OpenClaw 的思路是反过来的：假设 Agent 迟早会执行错误命令，然后用系统层限制把爆炸半径压到最小——纵深防御，而不是单点拦截。

## 做法：四道防线

**1. 文件系统白名单挂载。** sandbox 模式下，exec 和文件工具跑在容器里，宿主机只有显式 bind-mount 的 workspace 目录对它可见，读写权限分开声明。`~/.ssh`、`~/.aws`、系统目录默认根本不在沙箱视野内——Agent 想删也“看不见”。

**2. 非 root 运行 + 命令策略。** 容器内进程以普通用户运行，对未挂载路径没有写权限。exec 工具另有一层 pattern 策略：`rm -rf /`、`mkfs`、`dd of=/dev/*` 这类高危模式直接 deny，不依赖模型自觉。

**3. 高危操作人工审批。** 没被 deny 但属于破坏性的命令（删除、递归 chmod、覆盖写），可配置 approval 流程：命令先暂停，网关推送确认后才放行。MCP 写类工具同理，可单独设置 require-approval。

**4. 快照与回收站兜底。** workspace 在破坏性操作前自动打快照；文件删除默认移入 `.trash`，延迟真正清理。即使前三层全部失效，损失也限于一个可回滚的目录。

## 踩坑点

- 图省事把整个 `$HOME` 挂进沙箱，白名单退化成黑名单，隔离形同虚设。
- 挂载目录内的符号链接指向宿主机其他路径，构成逃逸通道；挂载前用 `realpath` 校验目录真实位置。
- 把 `docker.sock` 挂进沙箱“方便构建”，等于把宿主机 root 交出去。
- 觉得审批烦，全量关掉 approval，然后 Agent 一次“清理缓存”扫光构建产物。审批是高危命令的闸门，最多降级，不要关闭。
- 容器内跑 root：即使有挂载限制，权限放大依然危险，确认 user 映射真的生效。

## 可复用建议

1. 永远用白名单挂载，一个项目一个 workspace，不共享。
2. 破坏性操作遵循“先移动后删除”，trash 定期清理而非实时清空。
3. 保留审批与 exec 日志，事后能回答“谁在什么时候执行了什么”。
4. 做一次演练：在测试环境故意让 Agent 跑 `rm -rf`，验证 deny 规则、审批链路、快照回滚是否真实生效。没演练过的安全配置等于没配置。

## 总结

Agent 不会误删文件，不是因为模型聪明，而是因为架构假定它会犯蠢。OpenClaw 的 sandbox 模型用“白名单挂载 + 非 root + 命令策略 + 审批 + 快照”把单次错误的代价压到可回滚的范围内。安全配置的目标从来不是阻止 Agent 做事，而是让失败变得便宜。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/88f6b045333d617f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/5cdaee007309f3cf.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/1c3593e12c05fc43.png)

