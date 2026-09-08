---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 36642
source: 综合讨论
publishedAt: 2026-09-08
---

## 背景

Agent 拿到 shell 工具之后，最大的风险不是它不够聪明，而是它太"听话"——幻觉、提示注入、或者一个没校验的变量，都可能让它执行 `rm -rf` 这类不可逆操作。OpenClaw 的设计前提很明确：不假设模型永远正确，而是假设它终将犯错，然后在错误发生时把爆炸半径限制住。

## 问题

一条典型的翻车路径：用户说"清理一下构建产物"，模型生成 `rm -rf $BUILD_DIR/*`，而 `$BUILD_DIR` 在新开的 shell 里是空的，命令实际变成 `rm -rf /*`。当 agent 和你同权限、同文件系统时，这种错误没有任何拦截层，事后只有后悔。

## OpenClaw 的四层做法

**第一层：进程隔离。** 默认 sandbox 模式下，agent 的 exec 工具运行在容器内：非 root 用户、裁剪掉大部分 Linux capabilities、收紧的 seccomp profile。宿主机文件系统默认不可见，模型就算执行 `rm -rf /`，删的也只是容器自己的 overlay 层。

**第二层：最小挂载。** 只有 workspace 目录以 bind mount 方式进入容器，且可按需设为 read-only。agent 的可见世界就是你给它的那一个目录——"看不见"是最便宜的权限控制。

**第三层：工具策略与审批。** gateway 在工具调用链路上有 policy 层，可以按工具、按命令前缀做 allow/deny，对高危模式（递归删除、`dd`、`mkfs`、`chown /`）要求人工 approve 或直接拒绝。策略匹配发生在命令真正执行之前，prompt 劝不动它。

**第四层：审计与可回滚。** 每次 exec 调用连同完整命令、退出码落日志；workspace 建议初始化为 git 仓库，删除在版本控制里只是又一次 commit。

## 配置步骤（可复现）

1. gateway 配置中将 agent 执行环境设为 sandbox（Docker）模式；
2. 仅挂载 workspace，其余路径不进容器；
3. tools policy 中 deny 递归删除类模式，exec approval 对白名单外命令生效；
4. 容器内以非 root 运行，启用默认收紧 profile；
5. workspace 初始化 git，作为最后兜底。

## 踩坑点

- 为图方便把整个 home 挂进容器：第一、二层防线直接塌掉；
- 把 docker.sock 挂进 sandbox：等于给 agent 发宿主机 root，这是最常见的"自制后门"；
- workspace 里的软链接指向宿主机真实文件：容器内删除会穿透，挂载前先检查 symlink；
- 审批疲劳：approval 弹窗太频繁，人会无脑点 yes，策略应收敛到真正高危的操作；
- sandbox 不是备份：容器挡得住误删宿主机，挡不住误删挂载进来的 workspace 本身，快照仍然必须。

## 可复用建议

- 权限默认最小化，需要时再放开，不要反过来；
- 破坏性操作约束放在策略层，不要靠 prompt 里写"请小心"——提示词是建议，策略才是约束；
- 每个 agent 独立 workspace，避免交叉误伤；
- 定期演练：故意让 agent 执行一次指向临时目录的 `rm -rf`，观察它落在哪一层被拦截。

## 总结

OpenClaw 不指望模型不犯错，而是默认它一定会犯错：容器隔离限制可见范围，最小挂载限制可触碰范围，工具策略限制可执行动作，版本控制兜底可恢复性。四层各自独立，任何一层失守，下一层还在。对自建 agent 的同学来说，这套"假设失败、限制半径"的思路，比任何单点配置都值得抄走。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/bd51202918edf6c3.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/fcf11b3c49f97162.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/756f5d0d06ed34c9.png)

