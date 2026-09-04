---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 36044
source: 综合讨论
publishedAt: 2026-09-04
---

## 背景

OpenClaw 的典型用法是让 Agent 直接拿到工具：读写文件、跑 shell、调 MCP。能力一旦给出去，"模型很聪明，不会乱删"就不再是可接受的安全假设。真实事故形态都很朴素：glob 写宽了、`git clean -fdx` 顺手执行、README 里的注入指令让 Agent"清理构建产物"、路径解析漏了符号链接。删掉的不是测试目录，而是真实工作区。

所以 OpenClaw 的设计立场很明确：**模型的判断力只是安全模型里的一层，不是边界本身**。防误删不靠 prompt，靠系统级强制。

## 问题拆解

Agent 误操作有三个来源，对应不同的防护位置：

1. **模型自己犯错**：路径幻觉、把相对路径当绝对路径处理；
2. **外部注入**：仓库文件内容被当成指令（提示注入）；
3. **工具链放大**：一条看似无害的命令（`find -delete`、`chmod -R`）级联出大范围破坏。

如果只在 orchestrator 里做字符串检查，三个来源都可能绕过。所以 OpenClaw 把检查下沉到执行层。

## 分层做法

**第 1 层：workspace 根 + 路径规范化。** 所有文件工具拿到路径后先做 realpath 解析（含符号链接展开），再判断是否落在 workspace 根内。`..`、软链逃逸、大小写差异都在这层被拦下。

**第 2 层：能力显式声明，默认拒绝。** 会话启动时按配置生成工具清单；写、删、网络、exec 都是要单独授予的 capability。没声明 delete 能力的会话里，删除类调用在工具层直接报错，模型根本拿不到执行机会。示意配置（字段名以你本地版本为准）：

```toml
[sandbox]
root = "./workspace"
network = "deny"

[sandbox.capabilities]
write  = ["workspace/**"]
delete = "trash"        # 不真删，移入 .trash，保留 N 天
exec   = ["node", "python3", "git"]
```

**第 3 层：执行隔离。** shell 不直接跑在宿主环境。Linux 下走 bubblewrap + seccomp，macOS 下走 Seatbelt profile：只有 workspace 以读写方式挂进沙箱，其余路径不可见或只读；子进程自动继承，MCP server 作为子进程拉起时同样被包住。

**第 4 层：高危审批 + 全量日志。** 命中策略清单的命令（递归删除、改权限、git 破坏性子命令）必须人工确认；所有工具调用落 journal，出事能回放。

**第 5 层：MCP 注解接入策略。** MCP 的 `readOnlyHint` / `destructiveHint` 被映射到本地策略——注解只是提示、不可全信，但破坏性标注的工具一律再过一道确认。

## 踩坑点

- **符号链接**：workspace 里一个指向 `$HOME` 的软链就能击穿路径检查，必须先 realpath 再判界。
- **denylist 思维**：黑名单永远列不全，`find ... -delete`、`xargs rm` 变体无穷。必须是默认拒绝 + 白名单。
- **只包顶层进程**：沙箱不随 fork/子进程继承等于没做，MCP server 再 spawn 的进程也要在同一个 namespace 里。
- **路径规范化差异**：macOS 默认大小写不敏感、Unicode NFC/NFD 不一致，都会造成"检查时是 A、执行时是 a"。判断要在执行器内部做，避免 TOCTOU。
- **workspace 根给太大**：把 `$HOME` 当 root，沙箱就成了装饰。一个项目一个 root。
- **凭据泄漏**：`.env` 挂进沙箱等于送给模型，secrets 应在调用时注入且对文件工具不可见。

## 可复用建议

1. 读写分离：读可以宽，写必须窄。Agent 大部分价值在读，风险集中在写。
2. 删除一律走 trash 语义 + 保留期，给"撤销"留后路。
3. 写入走 staging/overlay：先落影子目录，diff 确认后应用，别让 Agent 直接改原文件。
4. 给沙箱配一个"对抗测试集"：一组必须失败的提示词和命令，进 CI，每次升级后跑一遍。
5. 把 MCP 工具注解当输入而不是真相，策略以本地配置为准。

## 总结

"Agent 不会误删文件"不是关于模型智商的命题，而是关于系统设计的命题。OpenClaw 的答案是五层叠加：路径规范化挡逃逸、能力声明挡越权、执行隔离挡进程级破坏、审批与日志兜底可审计、MCP 注解接入统一策略。每层单独看都不新鲜，叠起来之后，模型的错误才从"事故"降级成"日志里一条被拒绝的调用"。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/772bcfb42fbdb8b8.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/79e005b191430fad.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/36dbe4735a3ba7d8.png)

