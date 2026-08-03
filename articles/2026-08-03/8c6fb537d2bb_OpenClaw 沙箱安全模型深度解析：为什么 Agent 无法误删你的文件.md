---
title: OpenClaw 沙箱安全模型深度解析：为什么 Agent 无法误删你的文件
feedId: 31452
source: 综合讨论
publishedAt: 2026-08-03
---

# OpenClaw 沙箱安全模型深度解析：为什么 Agent 无法误删你的文件

## 背景：当自动化脚本获得“写文件”能力

随着 Agent、MCP 工具和自动化插件的大规模落地，工程师开始让 LLM 直接操作文件系统——创建目录、生成配置、清理临时文件，甚至读取密钥。这带来一个直白的担忧：**如果 Prompt 被污染，或者模型幻觉触发了一条 `rm -rf /`，我的实机数据会不会被清空？**

打开 OpenClaw 的讨论区，关于文件安全的提问一直居高不下。很多人以为给 Agent 挂一个 `/mnt/data` 只读就能高枕无忧，但实际工程中，权限控制远比这个复杂。本文依据 OpenClaw v2.3+ 的 sandbox 实现，梳理其安全边界，解释为什么在默认配置下，Agent 根本碰不到你的真实文件系统，更不用说误删。

## 问题：Agent 的文件操作到底有多危险？

假设你搭建了一个自动化环境，需要 Agent 协助处理日志文件：读取 `/var/log/app.log`，异常时生成一份报告写入 `/reports/`，然后清理原始日志。一个看似无害的 tool call 背后，可能出现以下风险路径：

1. **路径逃逸**：模型误将 `rm -rf /var/log` 当作清理指令执行。
2. **符号链接攻击**：用户将 `/etc/passwd` 软链到 `/mnt/logs/`，Agent 删除文件时意外销毁系统数据。
3. **插件越权**：某 MCP 插件声明只读文件能力，却因为实现缺陷获得了写入任意路径的权限。

传统方案是直接在宿主机运行 Node.js 进程并挂载真实路径，依赖 `file.path` 白名单做拦截，但这种白名单极易被路径组合绕过（如 `../../`）。OpenClaw 的解决思路不同——直接从内核层面隔离，让 Agent 无法触摸宿主机文件树。

## 做法：OpenClaw 的双层沙箱机制

OpenClaw 的 sandbox 由两层防护构成：**容器隔离层** 与 **只读覆盖层**。

### 1. 容器级隔离

OpenClaw 默认使用基于 Linux namespace 和 cgroup 的轻量容器（调用引擎为 bubblewrap 或 Docker 兼容层），而不是简单的 `chroot`。每次启动 Agent session 时，OpenClaw 会创建一个独立的名字空间：

- 独立 `mount namespace`，宿主机文件系统不可见
- 独立 `PID namespace`，Agent 进程无法看到或信号控制宿主机进程
- 受限 `network namespace`，可按需断开外网

所有 Agent 执行的文件操作都发生在容器内部视图里，天然无法引用宿主路径，即使硬编码 `/etc/shadow`，看到的也是容器内那个“空壳”文件树。

### 2. 复制即写 (read-only bind mount + tmpfs overlay)

为了让 Agent 能读取必要的配置文件，OpenClaw 支持将宿主机目录 **只读绑定**到容器内特定位置。例如：

```yaml
sandbox:
  mounts:
    - hostPath: /opt/configs
      containerPath: /sandbox/configs
      mode: ro
```

关键设计在于写操作的处理：Agent 请求写入 `/sandbox/output/` 时，并不会直接落到宿主对应的路径上。OpenClaw 在容器内使用 **overlayfs**，将只读层与一个临时可写层（tmpfs）叠加。任何“修改”或“删除”实际上发生在 tmpfs 临时层，容器销毁后该层自动丢失，只读挂载点原封未动。

也就是说，Agent 执行 `rm -rf /sandbox/configs/*`，仅是清空了 tmpfs 中的带写覆盖，下次挂载时 configs 依然完好。这便是**为什么 Agent 不会误删宿主文件**的核心原因。

### 3. 白名单限制写入口

对于确实需要持久化写出的场景，OpenClaw 要求在 sandbox 配置中显式声明可写挂载点，并将其映射到宿主某个专有目录。例如：

```yaml
sandbox:
  writableMounts:
    - containerPath: /sandbox/output
      hostPath: /data/agent_output/session_123
```

Agent 只能向 `/sandbox/output` 写入，其余位置一律只读或不存在。文件描述符操作、`open()` 调用都受此限制，即使在脚本中拼接路径，也无法逃逸到别的可写区。

## 踩坑点：信任边界容易被无意打破

尽管模型本身很坚固，实施中仍有几个常见失误会降低安全水位：

1. **将宿主机根目录整体挂载为只读**  
   有些用户为图省事，设 `hostPath: /` 进入容器。这不会直接造成文件被删（仍然是只读），但暴露了所有敏感文件的读取路径，如 `/root/.ssh/id_rsa`。Agent 如果被诱导读取，就会造成信息泄露。

2. **可写区指向重要目录**  
   把 `writableMounts` 的 `hostPath` 设置为 `/home/user` 等于把真实数据暴露给 Agent 写入。务必使用专门的沙箱目录，并配合磁盘配额。

3. **第三方 MCP 插件请求挂载额外路径**  
   部分插件会要求在容器内挂载宿主机可写路径以实现缓存或安装依赖。审核插件 manifest 中声明的 `mounts` 权限，避免让不受信任的插件获得写容器的机会。

4. **忘记设置容器资源上限**  
   即使文件系统安全，Agent 也可能会在 tmpfs 中写入大量垃圾文件消耗内存（因为 tmpfs 用内存承载）。提前配置 `tmpfsSize: 256m` 防止资源耗尽。

## 可复用建议：如何在不牺牲便利性的同时加固安全

在实际团队中推行 Agent 落地时，建议遵循以下工程实践：

- **最小挂载原则**：一个 session 只读挂载不超过 3 个必要的配置/数据目录，其余一律不进入 sandbox。
- **可写区 session 级隔离**：每次启动用 UUID 生成唯一的 `hostPath` 子目录，如 `/data/agent-output/{sessionId}/`，事后由 cron 定期清理，避免 session 间污染。
- **仅通过白名单 MCP 工具暴露文件能力**：不直接给 Agent 提供 `shell_exec` 这类任意命令执行工具，而是用封装好的文件操作工具，其实现内部强制限定 rootPath 为沙箱可写区，杜绝路径拼接绕过。
- **审计与录屏**：开启 OpenClaw 的 session 回放，遇到异常删除操作（即使是沙箱内）可以复现问题，反向优化 prompt 和工具描述。

## 总结

OpenClaw 的 sandbox 安全模型并不是在应用层做路径检查的“补丁”方案，而是从根本上利用 Linux namespace + overlayfs 构建了一个宿主文件系统无法触达的运行环境。Agent 看到的“文件删除”只是 tmpfs 临时层的覆盖动作，容器一销毁就了无痕迹。误删文件在这个架构下需要同时击破容器隔离、打破只读挂载、绕过可写区白名单——这在默认配置下几乎不可能发生。

理解了这层机制后，你就能更放心地把枯燥的文件整合、日志分析任务交给 Agent，而把精力放在真正需要人决策的事情上。安全的事，交给 sandbox 就好。

---

