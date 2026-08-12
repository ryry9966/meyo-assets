---
title: OpenClaw 沙盒安全模型：Agent 文件操作的隔离原理与防误删配置指南
feedId: 32706
source: 综合讨论
publishedAt: 2026-08-12
---

# OpenClaw 沙盒安全模型：Agent 文件操作的隔离原理与防误删配置指南

## 背景：当 Agent 拥有文件系统访问权

在 OpenClaw 这类 Agent 工作流中，我们常常让 LLM 调用工具来读写文件——保存中间结果、处理用户上传、甚至修改配置文件。一旦 Agent 拥有 `write_file` 或 `execute_code` 能力，误删系统文件的风险就会从“理论可能”变成“迟早会遇到”。传统的解决思路是权限控制（只给特定目录写权限），但在复杂自动化里，Agent 往往需要临时创建、删除大量临时文件，仅靠权限位很难覆盖所有边界情况，比如路径穿越、`rm -rf ../` 等。

真正的安全模型需要回答一个问题：**如何在让 Agent 完整操作文件系统的同时，保证宿主机不受任何持久性破坏？** OpenClaw 给出的答案是：每个 Agent 运行在一个基于 overlay 文件系统的临时沙盒里。

## 问题：为什么“只读挂载”还不够

很多早期实践会把关键目录以只读方式挂载给 Agent，其余目录可读写。但这类做法存在几个硬伤：

- Agent 可以删除可写目录下的文件，而这些目录往往是工作区的投影，可能包含宿主机的真实数据。
- 依赖绝对路径配置，一旦 Agent 使用了符号链接或 `..` 逃逸，保护就失效。
- 状态难以清理：Agent 运行后残留文件会干扰后续任务，甚至泄漏信息。
- 一旦 Agent 被注入恶意指令（prompt injection），破坏半径会扩大到整个可写区域。

我们需要一种“写时复制”隔离：Agent 看到完整的文件系统，但任何修改都只写在一个临时层上，不影响原始文件。Linux 内核的 OverlayFS 正好提供了这种原语。

## 做法：OpenClaw 的 Mount Namespace + Overlay 隔离

OpenClaw 使用了 Linux mount namespace 与 OverlayFS 的组合构造每个 Agent 的文件系统视图。其基本原理如下：

- **Lowerdir（只读层）**：宿主机的关键目录，如 `/usr`、`/etc`、用户指定的数据目录，以只读方式作为 lowerdir。
- **Upperdir（可写层）**：一个临时目录（通常在 `/tmp/openclaw-sandbox/agent-xxxx/upper`），所有修改和新建文件都在这里。
- **Workdir**：OverlayFS 要求的内部工作目录。
- **Merged（合并视图）**：Agent 看到的完整文件系统，由 lowerdir 和 upperdir 合并而成。读取时优先 upperdir，写操作全部落入 upperdir，删除操作通过在 upperdir 创建一个“白化文件”（whiteout）来屏蔽下层。

这样，即使 Agent 执行 `rm -rf /etc/passwd`，实际效果只是在 upperdir 里写入一个 whiteout 文件，宿主机 `/etc/passwd` 丝毫未改。Agent 退出后，OpenClaw 可以简单地丢弃整个 upperdir 和工作目录，所有痕迹消失。

### 关键配置示例

在 OpenClaw 的 Agent 配置中，可以这样声明沙盒：

```yaml
sandbox:
  enabled: true
  root:
    read_only: /host/etc:/etc
    read_only: /host/usr:/usr
    read_only: /host/opt/data:/data:ro   # 珍贵数据目录只读挂载
  writable_mount:
    - source: /tmp/openclaw-sandbox/upper
      target: /sandbox-write
      type: overlay
      options: lowerdir=/host,upperdir=/tmp/openclaw-sandbox/upper,workdir=/tmp/openclaw-sandbox/work
  ephemeral: true          # Agent 退出后自动销毁上层
  tmpfs_size: 512m         # 避免写满磁盘
```

启动 Agent 时，OpenClaw 会调用 `unshare --mount` 创建新 mount namespace，然后按上述配置挂载 overlay。对于不支持 pattr 嵌套的环境（如 Docker 容器内运行 OpenClaw），会自动降级为 tmpfs + bind mount 组合，但防误删的隔离逻辑不变。

## 踩坑点与排障

1. **OverlayFS 嵌套限制**  
   如果宿主机本身就在一个 overlay 文件系统上运行（例如 Docker 镜像层），需要内核版本 >= 5.11 才支持 overlay over overlay。老内核会报 `Too many levels of symbolic links`。解决办法：升级内核，或改用 tmpfs 模式。

2. **文件锁与 rename 原子性**  
   某些文件操作（如 sqlite 数据库写入）依赖跨层的锁行为，上层文件可能因 whiteout 机制导致锁失效。建议将需要频繁随机写的目录直接挂载到 upperdir 可写区，而不是通过 lowerdir 代理。

3. **白化文件堆积**  
   如果 Agent 频繁删除、创建同名文件，上层会积累大量 whiteout 节点，导致 merged 视图读性能下降。定期清理或使用 `ephemeral: true` 在任务结束后销毁上层即可。

4. **路径穿越绕过的残余风险**  
   即便有 overaly，如果 Agent 能直接访问宿主机 `/proc`，仍可能通过 `/proc/1/root` 逃逸。必须配合 `/proc` 和 `/sys` 的 remount 为只读或隐藏敏感子路径：

   ```yaml
   sandbox:
     proc:
       mount_options: "hidepid=2,subset=pid"
   ```

## 可复用建议

- **最小化 lowerdir**：不要将整个 `/` 作为 lowerdir，只暴露 Agent 任务明确需要的目录，减少信息泄露面。
- **数据目录只读挂载**：对于原始数据集、模型权重等，明确以 `:ro` 声明只读，即使 Agent 试图写入也会因 upperdir 没有对应写权限而失败。
- **使用 tmpfs 存储临时文件**：将 `/tmp`、`/var/run` 等作为独立的 tmpfs 挂载，避免 Agent 写满磁盘宿主机，并加快 IO。
- **日志审计不可省**：在沙盒外部开启 `auditd` 跟踪所有 `unlink`、`rmdir` 调用，结合 OpenClaw 的 Agent ID 可回溯每一次删除操作。
- **自动化清理脚本**：对于非短暂运行的 Agent（`ephemeral: false`），用 cron 定期清理超过 24 小时的 upperdir，防止 inode 耗尽。

## 总结

OpenClaw 的沙盒安全模型不依赖 Agent 的“守法意识”，而是利用 Linux 命名空间和叠加文件系统，从内核层面保证所有文件更改都是可撤销、可丢弃的。误删不再成为运维灾难，本质上把文件操作从“直接穿透宿主机”变为“只在一次性的临时层上表演”。对于任何需要在自动化流程中赋予 Agent 文件写权限的场景，这种设计都是目前工程上成本最低、可靠性最高的选择。

实施时，理解 overlay 的上下层语义、处理好内核兼容性、以及做好资源限制，就能让 Agent 在安全边界内充分发挥文件操作能力，而不必战战兢兢地限制它的每一条指令。

---

