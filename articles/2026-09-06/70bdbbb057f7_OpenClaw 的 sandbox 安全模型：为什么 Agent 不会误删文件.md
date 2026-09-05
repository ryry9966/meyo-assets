---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 36240
source: 综合讨论
publishedAt: 2026-09-06
---

## 背景

给 Agent 接上 shell 之后，有个绕不开的问题：它手里有 `rm` 的执行权，你凭什么放心。在提示词里写“请不要删除文件”不构成安全边界——模型会犯错、会误解指令、也可能被注入内容带偏。OpenClaw 的思路很明确：安全问题用系统设计解决，不靠模型自觉。

## 为什么不会误删：三道闸门

OpenClaw 的 sandbox 不是单点开关，而是叠起来的三层。删文件这类操作要连续穿过三道闸门，才可能触达你的真实数据。

**第一道：文件系统边界。** Agent 默认只挂载 workspace 一个目录。沙箱里看到的世界就是 workspace 加系统目录，你的家目录、代码仓库、照片库根本不在它的文件系统视图里。它不是“不被允许删”，而是“看不见”。

**第二道：容器隔离。** 开启 Docker sandbox 后，会话跑在隔离容器内：非 root 用户运行，文件系统可丢弃，workspace 以 bind mount 挂入，且是唯一持久化路径。即使 Agent 在容器里执行了破坏性命令，影响的也只是沙箱内部状态，宿主机其他路径无从谈起。

**第三道：工具策略与审批。** exec 工具可以配置 allowlist 和敏感命令审批（ask 模式）。命中规则的命令先弹给你确认，拒绝即终止。路径校验发生在工具层：越出 workspace 的写删请求直接报错，模型收到的是结构化拒绝反馈，而不是静默失败。

## 实际配置步骤

1. 在 `openclaw.json` 的 sandbox 段把 mode 设为 `non-main` 或 `all`；
2. scope 按需选 `agent` / `session` / `shared`，多 Agent 各自隔离建议用 `agent`；
3. 检查 docker 段的挂载配置，确认 workspace 是唯一 bind mount，只读场景改成 ro；
4. 给 exec 开 ask 审批，把 `rm`、`dd`、`mkfs` 这类模式加进敏感清单；
5. 用无害命令探测验证：让 Agent `ls /`，确认它看到的只是沙箱视图。

## 踩坑点

- 图方便把整个家目录挂进沙箱，第一道边界直接失效，后两道挡不住“合法路径上的合法删除”。
- 容器里以 root 跑，误操作和逃逸的成本都变低。
- 审批弹窗嫌烦就关掉，等于只剩文件系统边界单点防御。
- 多 Agent 共享同一个 workspace，互相覆盖或清理。这不是安全漏洞，但结果也是“文件没了”，排查时容易误判成沙箱失效。
- 网络不限制，Agent 能 curl 脚本再执行，等于自己给自己开后门。

## 可复用建议

- 把 sandbox 当默认态而不是可选优化：新 Agent 一律先跑在隔离环境，确认行为再放宽。
- 验证边界用“探测”而不是“信任”：定期让 Agent 尝试越界操作，确认被拒并留下日志。
- 审批清单按命令模式维护，而不是逐条命令——单拦一条 `rm -rf` 没有意义。
- 审计日志留着：出问题先看工具层的拒绝记录，再查容器层，最后才怀疑模型。

## 总结

Agent 不误删文件，靠的不是它“很听话”，而是设计上让它接触不到、改不动、够不着你的数据。OpenClaw 的 sandbox 模型把“能力”和“权限”拆开：模型可以很强，边界由系统说了算。对做自动化的同学来说，这套分层思路比任何单个开关都值得搬走。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/9ad535f0d1e0e5e6.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/32da3329168d301a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/58800c7466c101b0.png)

