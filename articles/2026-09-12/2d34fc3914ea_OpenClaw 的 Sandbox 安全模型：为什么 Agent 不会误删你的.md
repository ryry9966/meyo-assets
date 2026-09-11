---
title: OpenClaw 的 Sandbox 安全模型：为什么 Agent 不会误删你的文件
feedId: 37104
source: 综合讨论
publishedAt: 2026-09-12
---

## 背景

OpenClaw 是常驻运行的 Agent 网关：它不只是陪你聊天，还拿着 exec、文件读写这类“真工具”，在你不在屏幕前时跑定时任务、响应消息。这意味着一个现实问题必须正面回答——模型哪天把 `rm` 的参数搞错了怎么办？“Agent 不会误删文件”不是对模型能力的承诺，而是对系统设计的结论。下面拆开讲它靠什么兜底。

## 问题：误删是怎么发生的

实际事故里，模型很少“故意删”，常见的失败模式是三种工程失误：

1. **路径解析错误**：相对路径、`~` 展开、软链接指向工作区之外；
2. **命令拼接错误**：变量为空，`rm -rf "$DIR"/` 退化成危险路径；
3. **任务理解偏差**：把“清理构建产物”执行成“清理整个目录”。

所以防线不能押注在模型自觉上，必须靠系统边界。

## OpenClaw 的四层防线

1. **工作区边界（workspace）**：文件类工具（read/write/edit/ls 等）被绑定在 workspace 根目录内，外部路径默认不可见、不可写。
2. **Docker sandbox**：shell 命令放进一次性容器执行，只 bind mount workspace。容器里根本看不到宿主的其他目录，`rm -rf /` 删的是容器自己的根。
3. **Exec 审批**：对高危命令模式（`rm -rf`、`sudo`、`mkfs`、`dd` 等）触发人工确认，审批请求直接发到你的消息渠道；命中拒绝规则的直接拦下。
4. **兜底**：workspace 纳入 git/快照 + 审计日志，最坏情况可回滚。

一句话总结：模型负责“想做什么”，sandbox 负责“能不能做到”。

## 实操步骤

1. 确认宿主机 Docker 可用，在网关配置中开启 sandbox 模式（可按 agent 粒度区分，主对话与后台任务策略可以不同）；
2. 显式把 workspace 设到专用目录，例如 `~/openclaw-workspace`，不要偷懒用家目录；
3. 配置 exec 审批：deny 列表放 `rm -rf`、`sudo`、`mkfs`、`dd`；ask 列表放删除、批量移动和写入类操作；
4. workspace 内 `git init` + 定时 commit，作为可回滚快照；
5. 做一次破坏性演练：让 agent 尝试 `cat /etc/passwd`、删除 workspace 外文件、对系统目录执行 rm，确认全部被拒或落在容器内。

## 踩坑点

- **workspace 设成 `~`**：边界等于没有，文件工具看似受限，实际覆盖了你全部个人文件。
- **只配审批不开 sandbox**：关键词审批可被绕过，比如 `bash -c "echo cm0gLXJmIC4= | base64 -d | bash"` 里根本不含 `rm` 字样。审批只是第二层，容器隔离才是根本。
- **挂载过宽**：图省事把整块数据盘 mount 进容器，等于把禁区搬进了 sandbox。
- **容器复用**：多个 agent 共享一个长驻容器会互相污染状态，也可能留下越权改动，优先一次性容器或按 agent 隔离。
- **版本差异**：sandbox 与审批的配置键名随版本可能调整，升级后先重跑第 5 步演练再上生产。

## 可复用建议

- **最小权限优先于智能约束**：能用 mount 边界解决的，不要靠 prompt 求模型“别乱来”。
- **纵深防御**：边界 + 隔离 + 审批 + 快照，任何一层失效都有下一层接着。
- 把破坏性演练写成脚本，纳入每次升级的验收流程。
- 审批提示里保留完整命令原文，人工确认时看命令本身，而不是看 agent 的解释。

## 总结

“OpenClaw 的 Agent 不会误删文件”是一个工程结论：文件工具锁在 workspace，shell 锁在容器，高危操作过审批，快照做最终兜底。Sandbox 模型的本质是把“信任模型”的问题转化为“划定边界”的问题——这比每次执行前默念“模型这次应该没问题”可靠得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/ea891f4d1bde858d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/9c14b31eac963c02.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/eb303c8404ced2ee.png)

