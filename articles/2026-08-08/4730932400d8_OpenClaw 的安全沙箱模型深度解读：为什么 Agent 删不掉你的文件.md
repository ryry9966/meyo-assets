---
title: OpenClaw 的安全沙箱模型深度解读：为什么 Agent 删不掉你的文件
feedId: 32094
source: 综合讨论
publishedAt: 2026-08-08
---

# OpenClaw 的安全沙箱模型深度解读：为什么 Agent 删不掉你的文件

## 背景：当 Agent 拿到文件系统这把“刀”

在 OpenClaw + Agent（或 MCP 插件）的工程实践中，只要给了 Agent 文件系统访问能力，立刻就会面对一个挥之不去的顾虑：**它会不会删掉宿主上的关键文件？** 即便提示词里写满“严禁 rm -rf /”，但在长上下文、多步推理、工具链嵌套的场景下，幻觉或误解难免发生。与其靠提示词乞求 Agent 自律，不如直接在系统层面让它“删不掉”。这正是 OpenClaw sandbox 安全模型的出发点。

## 问题拆解：我们要防的不是“恶意”，而是“不确定”

多数用户其实并不担心 Agent 故意作恶，而是担心以下三类情况：

1. **路径拼写错误**：Agent 想删临时的 `/tmp/output/old/`，结果参数拼成 `/tmp/output/../`，越界到父目录。
2. **提示注入导致意外操作**：上游数据被塞入“删除所有日志”的指令，Agent 照做。
3. **模型对文件系统语义理解偏差**：以为“删除缓存”是安全操作，实际映射到生产目录。

因此，安全模型的目标不是禁止删除操作，而是**将删除操作限定在一个不会造成持久化损害的沙箱里**。OpenClaw 的实现思路是把文件系统做分层隔离，让 Agent 看到的“世界”只是一个临时的、可丢弃的视图。

## 做法：基于 overlay 的分层文件系统隔离

OpenClaw 当前推荐的安全配置是启用 **sandbox overlay engine**，底层依赖 Linux 的 OverlayFS 和 namespace 机制。核心架构如下：

- **lower 层（只读基座）**：用户指定的安全基准目录，以只读方式挂载，Agent 可见但不可改。
- **upper 层（可写暂存）**：通常放在 tmpfs 或一个临时目录，所有写操作（创建、修改、删除）都落在这层。
- **merged 视图**：Agent 实际看到的文件系统，由 lower + upper 合并而成。
- **删除的本质**：当 Agent 执行 `rm /data/config.yaml` 时，如果该文件来自 lower 层，内核会在 upper 层创建一个 **whiteout 文件**，让 merged 视图里该文件“消失”。但 lower 层原文件毫发无伤。
- **新写入的文件**：直接写在 upper 层，Agent 进程退出后，若未声明持久化，整个 upper 层会被丢弃。

这套模型让 Agent **感觉自己在正常操作文件系统，实际所有破坏性动作都被困在沙箱 upper 层**。

### 配置示例（仅示意，具体以 OpenClaw 版本为准）

```yaml
sandbox:
  engine: overlay
  base_dir: /opt/data/readonly   # lower 层
  work_dir: /run/openclaw/upper  # tmpfs 上的 upper 层
  writable_dirs:
    - /workspace/output          # 允许持久化的目录，会 bind mount 到外部
  readonly_dirs:
    - /workspace/config
    - /workspace/models
  syscall_filter:
    allow_unlink: true           # 允许删除，但落在 sandbox 内
    block_mount: true
    no_new_privs: true
```

启动 Agent 时，OpenClaw 会创建一个新的 mount namespace，将 lower 和 upper 按上述规则组合，Agent 进程的根文件系统便成为一个受控的沙箱视图。

## 踩坑记录：沙箱不是银弹，这些细节容易翻车

1. **whiteout 造成的“伪删除”与 Agent 状态不一致**  
   如果 Agent 在任务中途删除了一个文件，检查 `ls` 或 `stat` 发现文件已消失，后续推理依赖“该文件已不存在”这一事实。但任务结束后 upper 层被销毁，下次启动 Agent 时文件又恢复原状。若使用同一 Agent 会话多次跨沙箱生命周期操作，会导致认知错乱。解决办法：要么每次任务使用全新的 upper 层，要么通过提示词明确告知“文件系统在会话间会重置”。

2. **上层耗尽导致写入失败**  
   upper 层放在 tmpfs，当 Agent 大量写入临时数据（比如解压大包、生成大量中间文件）时可能撑满内存。配置里务必设置 `size` 限制，并监控磁盘用量，最好配合 systemd-tmpfiles 做定期清理。

3. **符号链接与路径绕过**  
   如果 lower 层含有指向外部的符号链接，Agent 通过跟踪链接可能读写到 sandbox 外的文件。OpenClaw 的 syscall filter 默认会阻止跨 mount namespace 的链接跟踪，但若用户自己加入了 `writable_dirs` 且允许外部挂载，则需要格外小心，建议遵循“最小编写原则”，任何持久化目录只用 bind mount 挂入，而非符号链接。

4. **删除受保护目录的假象**  
   Agent 执行 `rm -rf /workspace/config` 时，因为是只读视图，内核返回 `EROFS`（Read-only file system）。如果 Agent 未正确处理该错误，可能错误地认为目录已删除，继续执行危险动作。因此 Agent 工程师应注意处理工具调用结果的返回值，并在提示词中明确“读取权限错误时停止操作”。

## 可复用的工程化建议

- **按任务划分配置**：数据预处理任务的 Agent 只挂载原始数据目录（只读）和一个可写的输出目录；分析类 Agent 则只给临时空间。不同任务用不同的 `sandbox.yaml` 模板。
- **持久化输出用专用挂载**：不要把整个 workspace 设为可写，只用 `writable_dirs` 精确指定 `/workspace/output`，并映射到宿主机的持久卷，这样即使 upper 销毁，产物也不会丢。
- **无网络加固**：如果任务不需要网络，在 sandbox 配置里关闭网络访问（`network_mode: none`），这样即使 Agent 试图通过远程下载恶意脚本，也无计可施。
- **配合 seccomp 剖面**：OpenClaw 的 syscall filter 允许精细控制，建议除了 `unlink` 外，禁用 `mount`、`pivot_root`、`kexec_load` 等高危系统调用。
- **日志审计**：在 Agent 工具回调层记录所有删除、修改操作日志，配合 sandbox 外部监控，对批量删除告警，即使无害也能留痕。

## 总结：把安全边界沉到内核层，而不是依赖提示词

OpenClaw 的 sandbox 安全模型通过 OverlayFS + mount namespace + syscall filter 这一组合，让 Agent 的文件删除操作变成一场“沙箱内的独角戏”。它不能替代权限管理和最小化授权，但确实把“误删文件”这类常见风险降低到了可控范围。工程上，理解 whiteout 和只读层的语义、合理规划可写目录、做好会话生命周期管理，才能真正发挥 sandbox 的防护价值。

在 Agent 自动化的路上，下一次当你犹豫要不要给 Agent 文件写入权限时，不妨先打开 OpenClaw 的 sandbox —— 你会发现，**不让它删到，比不让它删，可靠得多**。

---

