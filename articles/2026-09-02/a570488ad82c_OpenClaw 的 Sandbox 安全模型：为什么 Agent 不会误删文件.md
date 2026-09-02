---
title: OpenClaw 的 Sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 35807
source: 综合讨论
publishedAt: 2026-09-02
---

## 背景

Agent 拿到 shell 和文件工具之后，最怕的不是它不会干活，而是它"自信地干错活"：路径解析错一位、通配符展开失控，`rm -rf` 就可能落在真实目录上。OpenClaw 的设计前提不是"相信模型"，而是假设模型一定会犯错，然后在架构上把单次错误的代价压到最小。

## 问题：命令黑名单挡不住所有删除

社区早期最常见的方案是拦截 `rm -rf`、`git clean -fd` 这类命令。但这在工程上站不住脚：

- `find . -delete`、`python -c "import shutil; shutil.rmtree('.')"`，甚至写个临时脚本再执行，都能绕过字符串匹配；
- 就算拦住了 shell，MCP 工具和第三方插件如果直接跑在宿主机上，照样能碰真实文件系统。

结论很直接：**过滤只能做辅助，隔离才是底线**。

## OpenClaw 的做法：四层防线

1. **文件系统隔离**。Agent 的工具调用默认发生在容器沙箱内，工作区是沙箱内部目录；宿主机目录要么不挂载，要么只读 bind-mount。Agent 看到的和真正落盘的是两个世界，这是最硬的一层。
2. **工具级策略引擎**。文件工具不直接落地，先过策略校验：路径 allowlist 限定在工作区内；resolve 符号链接后再判断，防 symlink 逃逸；破坏性操作默认要求显式确认，或者干脆不暴露 delete 工具。
3. **MCP / 插件按会话授权**。每个 MCP server 声明所需 scope，插件进程也跑在沙箱里，拿最小权限。第三方工具想删宿主机文件？它根本没有那个视图。
4. **写前快照**。用 overlayfs 或 git 对工作区打快照，批量写之前先留还原点，出问题回滚即可。

## 实践步骤

- 启动时显式设置 workspace root，不要依赖默认值；
- sandbox 设为强制模式，并在 CI 里加检查：Agent 进程用户对工作区外路径无写权限；
- 高危操作（跨目录批量删除、覆盖未跟踪文件）配置 human-in-the-loop；
- 定期用对抗性 prompt 自测，比如"把 /tmp 和项目目录都清一下"，看它实际能摸到哪。

## 踩坑点

- **真实项目目录以读写方式直接挂进沙箱**，等于没隔离，这是最常见的配置错误；
- **符号链接逃逸**：Agent 在工作区里 `ln -s /important/dir` 再删目标，必须 resolve 后校验或禁止建链；
- **确认疲劳**：全都弹确认，用户会无脑点允许；门槛要只留给真正的高危动作；
- **快照无限增长**：不配清理策略，跑一周磁盘就满。

## 可复用建议

- 隔离优先于过滤，命令黑名单只是锦上添花；
- 缩小爆炸半径：沙箱内不放密钥、不挂 credentials、不给 sudo；
- 默认拒绝：allowlist 远比 denylist 可靠；
- 每接入一个新工具就问一句：它能不能绕过文件工具，直接碰到底层文件系统？

## 总结

"Agent 不会误删文件"从来不是模型能力问题，而是系统设计问题。OpenClaw 用"隔离为主、策略为辅、快照兜底"的组合，把单次错误锁死在沙箱里。任何打算让 Agent 碰真实文件的团队，都可以直接搬这套思路：先划边界，再谈能力。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/1fb430ba381067df.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/a0c9bd10382c4afd.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/aea40ad3ad246cf9.png)

