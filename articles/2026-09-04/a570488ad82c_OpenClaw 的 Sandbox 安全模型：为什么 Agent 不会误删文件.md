---
title: OpenClaw 的 Sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 36066
source: 综合讨论
publishedAt: 2026-09-04
---

## 背景

OpenClaw 的 Agent 默认拿到的是一套“全权”工具：exec、文件读写、跑脚本。很多刚上手的用户第一反应是担心——模型哪天抽风，一条 `rm -rf` 把家目录清了怎么办？这篇帖子拆一下 OpenClaw 的 sandbox 安全模型，说清楚一件事：防误删靠的不是模型听话，而是架构边界。

## 问题：错不在“会不会”，在“错得起吗”

LLM 的错误是概率性的：路径幻觉、对指令的过度推断、把“清理缓存”理解成“清理目录”，都可能发生。如果工具直接跑在宿主机上，一次低概率错误就是不可逆损失。所以安全设计的第一原则不是“让模型别犯错”，而是“让它犯错时损失可控”。

## 做法：三层边界叠加

OpenClaw 的沙箱模式大致是三层防线（具体 config 键名以你实际版本为准）：

1. **进程隔离**。把 `agents.defaults.sandbox.mode` 设为 `all` 后，exec 类工具不再直接在宿主机执行，而是落到 Docker 容器里。容器有自己独立的文件系统视图，Agent 执行 `rm -rf /`，删掉的是容器内的目录，宿主机毫发无伤。
2. **文件系统最小挂载**。只有 workspace 以 bind mount 方式进容器，其余宿主路径对 Agent 不是“被保护”，而是“不存在”。这是关键差别：权限可以被绕过，不存在的路径无从谈起。
3. **工具策略与确认门**。tool policy 控制哪些工具可用，exec approvals 对高危命令要求宿主侧人工确认，elevated 操作默认不放行。

验证方法很简单：起一个 agent，让它 `ls /`，再让它尝试删除一个宿主机上存在但未挂载的文件——返回的应该是 not found 而不是删除成功。顺手 `docker ps` 确认命令确实跑在容器里。五分钟就能做完，建议每个人都做一次。

## 踩坑点

- 图省事把 `/` 或整个 home 目录挂进沙箱，等于亲手把隔离拆了。挂载列表保持最小。
- 沙箱保护的是 workspace **之外**的宿主机；workspace 本身是双向挂载，Agent 照样删得掉。别把沙箱当备份。
- 把 exec approvals 全部设成 auto-approve，防线就只剩容器这一层。多层防线要同时生效。
- macOS/Windows 上沙箱依赖 Docker，路径映射、符号链接、大小写敏感都有坑。第一次配置建议从 Linux 或 WSL2 起步，跑通再回桌面系统。

## 可复用建议

- 一句口诀：**默认沙箱、最小挂载、高危确认、workspace 进 git**。
- 每个 agent 用独立的 sandbox scope，避免多 agent 之间状态串味。
- 做一次“破坏性演练”：故意让 Agent 执行激进的删除命令，亲眼观察边界在哪，比读十遍文档都有效。
- 定期审计 tool policy。插件和 MCP server 带进来的新工具，是沙箱边界外最大的变量。

## 总结

Sandbox 的价值在于把“模型可能犯错”从安全问题降级成运维问题：容器重建即恢复，损失被限定在显式挂载的 workspace 内。与其信任模型的对齐，不如信任文件系统的“不存在”——这才是 OpenClaw 敢让 Agent 拿 shell 的底气。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/1af628936cdd4594.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/bd6cb939b6cde3cb.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/818dfa3bb95d9ad3.png)

