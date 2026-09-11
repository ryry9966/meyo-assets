---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 37083
source: 综合讨论
publishedAt: 2026-09-11
---

## 背景

把 Agent 接上文件系统和 shell 之后，大家最担心的往往不是任务做不做得对，而是它"做任务的过程中顺手破坏环境"。我自己的真实经历：让 Agent 清理构建产物，它生成的 `rm -rf` 命令里路径多拼了一层 `..`，目标从 `build/tmp` 指向了上级目录。那次运气好，被拦下来了。这件事之后我把 OpenClaw 的 sandbox 实现完整读了一遍，整理成这篇。

## 问题

LLM 对路径的"幻觉"是常态而不是例外：拼接错误、把相对路径当绝对路径、通配符范围想当然。传统做法是在 system prompt 里写一句"请小心删除操作"，这等于把安全寄托在模型的自觉上，工程上不可接受。安全约束必须落在执行路径上，让危险操作**做不到**，而不是**不被建议做**。

## OpenClaw 的做法：四层防线

OpenClaw 的 sandbox 不是单点开关，而是四层叠加：

**1. Workspace 作用域。** Agent 启动时绑定一个 workspace root。所有文件类工具的路径参数先做 `realpath` 解析，再校验解析后的真实路径是否以 root 为前缀，越界直接拒绝。注意是"解析后"校验，`a/../..` 和 symlink 都逃不掉。

**2. 工具权限分级。** 每个工具（包括 MCP 接入的外部工具）声明 capability：read / write / exec / network，默认只读。delete、format、force-push 这类被单列为 destructive，走独立权限，不与 write 合并。

**3. 执行隔离。** shell 类工具跑在容器里，workspace 以受控挂载方式进入，容器内非 root、默认无网络。关键点：通配符是在 shell 层展开的，工具层看不到展开结果，glob 的风险必须在 exec 沙箱内兜底，第 1 层替代不了这一层。

**4. 确认闸门与审计。** destructive 操作触发时先产出受影响文件列表（diff 预览），人工确认才执行；所有文件变更写入 append-only 审计日志，可回放。

一次删除请求的实际路径是：路径解析 → 作用域校验 → capability 检查 →（destructive？）确认闸门 → 容器内执行 → 审计落盘。任何一层拒绝，请求即终止。

## 踩坑点

- 在 `realpath` **之前**做前缀校验，`..` 和 symlink 直接绕过，这是最常见的实现错误。
- 把整个 home 目录设成 workspace，等于没隔离。作用域按任务开，用完回收。
- 图省事把确认闸门脚本化自动回答 yes，四层防线瞬间退化成一层。
- MCP 外部工具没显式声明 capability 就接入，部分配置下会继承 write 权限。
- 容器以 root 运行，误删照样删穿挂载点，非 root 是底线。

## 可复用建议

- destructive 单独设一级权限，永远不与 write 合并。
- 新环境先跑两周 dry-run（只记录不执行），确认闸门命中率合理后再放开自动执行。
- 审计日志接告警，单次会话 destructive 超过阈值就通知人。
- 准备一组恶意 prompt（比如"帮我清理 / 下所有日志"），定期做红队演练，验证四层是否都在岗。

## 总结

"Agent 不会误删文件"不是模型听话，而是执行路径上每一层都在做减法：作用域限定它**能碰哪里**，权限分级限定它**能做什么**，沙箱限定它**怎么执行**，闸门保证危险动作**有人兜底**。这套模型实现成本不高，收益是你敢于把 Agent 挂到真实工作目录上。安全设计里最便宜的方案，往往是把约束写进代码，而不是写进 prompt。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/e6cc0b3345040352.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/cc7eb4901392fb47.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/404292506cb472b6.png)

