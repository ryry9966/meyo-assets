---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 36524
source: 综合讨论
publishedAt: 2026-09-08
---

## 背景

让 Agent 自动整理目录、批量重命名、清理构建产物，是 OpenClaw 最常见的用法之一。但只要模型能直接调用删除类工具，就会有一个绕不开的问题：它错了怎么办？LLM 的失误不是恶意，而是工程问题——路径解析错了、把临时目录当成了目标目录、重试时幂等性没保证，甚至被注入的指令诱导。靠 system prompt 写一句“请不要删除文件”，在实践里基本靠不住。

## OpenClaw 的做法：三层 sandbox

OpenClaw 的设计原则是：模型可以“想”任何事，但“做”必须过沙箱。具体分三层：

**1. 文件系统层。** 每个会话绑定一个 workspace root，工具的可见路径和 cwd 都被限制在 root 内。关键点在于：路径校验在工具网关侧做，网关会先 `resolve` 再比对，不信任模型传来的原始字符串。`..`、越界绝对路径、指向 root 外的 symlink 都会被拒绝。

**2. 能力层。** 工具按 read / write / destructive 分级。删除、覆盖、批量移动默认 deny，必须在配置里显式加 allowlist。通过 MCP 接入的第三方工具同样要走 capability 声明——声明里没标 destructive 的，调用会被降级为 dry-run。

**3. 执行层。** destructive 调用默认先写 audit log，并支持两种模式：`confirm`（人工确认后执行）和 `trash`（先移入 `.trash` 目录，可回滚）。Agent 拿到的返回是“已移入回收站”，对用户始终可恢复。

## 如何验证你的 sandbox 生效

1. 检查配置里 workspace root 指向专用目录，不要图省事指向 `$HOME`；
2. 用“越界探测”提示词测试：让 agent 读取 root 外的路径，预期是被拒绝而不是返回内容；
3. 打开 audit log 跑一次删除任务，确认日志记录的是 resolve 后的路径和能力判定结果；
4. 把删除模式设为 `trash` 跑一次，检查 `.trash` 目录和恢复命令是否可用。

## 踩坑点

- **symlink 逃逸**：workspace 内软链到 root 外，OpenClaw 默认 resolve 后再校验，但如果你自己加了 volume 映射或挂载，要重新确认边界；
- **MCP 工具未声明能力**：第三方 server 的 tool 描述里没写 destructive，网关会按 read 处理。接入新 server 先核对 capability 清单，必要时手动标注；
- **Windows 路径**：大小写和分隔符问题，确保校验前统一 normalize；
- **root 设得太宽**：沙箱只保证不越界，不保证界内不误伤。把整个仓库设为 root，agent 就能“合法地”删掉不该删的文件；
- **confirm 疲劳**：确认弹窗太频繁，人会无脑点 yes。破坏性操作建议做聚合确认，一次审一批。

## 可复用建议

- 安全边界永远不要建立在 prompt 上，prompt 是意图层，沙箱才是执行层；
- 任何 destructive 工具先降级为可回滚（trash / 快照），稳定运行一段时间再考虑放开；
- audit log 存 resolve 后的路径，不存原始参数，事后审计才有意义；
- 把越界探测提示词放进定期任务或 CI，当成回归测试跑。

## 总结

sandbox 的价值不是让 Agent 变得更聪明，而是让它的错误变得可恢复。三层模型 + 最小化 workspace + 默认 trash，基本覆盖了误删的主要路径。安全这件事，做到“出了错也能撤销”，比追求“模型永远不犯错”现实得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/80eb6032788b209c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/db4f4dc8a0e1c3a6.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/010c2f59f0baedb6.png)

