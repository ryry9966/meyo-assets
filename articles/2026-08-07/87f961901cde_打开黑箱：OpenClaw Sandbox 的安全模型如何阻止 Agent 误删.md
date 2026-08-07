---
title: 打开黑箱：OpenClaw Sandbox 的安全模型如何阻止 Agent 误删文件
feedId: 31996
source: 综合讨论
publishedAt: 2026-08-07
---

## 背景：当 Agent 握住了 `rm` 的权柄

无论你是用 OpenClaw 编排自动化任务，还是把 Agent 作为 MCP 服务器的执行终端，都绕不开一个核心担忧——**它会不会在某条错误指令下，把自己辛辛苦苦写好的配置文件、数据目录甚至系统文件删得干干净净？**

传统 DevOps 中的`rm -rf /`梗，在自治 Agent 场景下变成了真实威胁：LLM 会根据自然语言生成 shell 命令，而幻觉、上下文污染或 prompt 注入都可能导致毁灭性操作。许多 Agent 框架直接暴露宿主机文件系统，仅依赖“请小心一点”的提示词约束，这在工程中是不可能接受的。

OpenClaw 的选择是引入可配置、可审计的多层 sandbox，将 Agent 的执行范围收缩到受控边界内。本文拆解这套机制的核心设计，并给出可复现的配置方法。

---

## 问题定义：不是防“恶意”，而是防“意外”

需要明确，OpenClaw 的 sandbox 并不是针对高级持久化威胁（APT），它的设计目标是**防止 Agent 因错误推理或意外上下文而产生数据丢失**。典型场景包括：

- 清理临时文件时路径推导错误，删除项目代码
- MCP 插件调用返回恶意构造的命令建议，Agent 盲从执行
- 用户在 prompt 中误写了危险指令，而 Agent 未做二次确认

这些场景的共同点：Agent 本身无恶意，但需要工程约束来兜底。

---

## 多层防御模型拆解

OpenClaw 的 sandbox 由三个正交层叠加，即使单层在极端情况下被绕过，其它层仍能阻断危险操作。

### Layer 1 – 文件系统视图隔离

Agent 进程被限制在一个独立 mount namespace 中，根文件系统默认以只读方式挂载（`readonly rootfs`）。需要写入的位置通过显式配置挂载为 overlay 或 tmpfs。架构如下：

```
Host Root
├─ /workspace   (bind mount -> /data/safe-container/workspace, rw)
└─ /app         (原始 overlay, ro)

Agent 可见:
/ (ro)
├─ /app       (ro)
├─ /workspace (rw, tmpfs backed)
└─ /tmp       (rw, tmpfs)
```

Agent 执行 `rm -rf /` 时，`/app` 等只读目录会直接返回 `Read-only file system`，不会影响宿主机文件。破坏力局限在 `/workspace` 和 `/tmp` 这类被显式授权的可写区域，且这些区域可以通过磁盘配额限制大小，防止被恶意写满。

### Layer 2 – 命令危险等级评估

在系统调用拦截之前，OpenClaw 的 shell 执行器会对即将执行的命令做一层静态解析。配置文件 `sandbox.rules` 中定义了危险命令正则表：

```yaml
dangerous_commands:
  - \brm\s+-rf\b
  - \bshred\b
  - \bdd\s+if=.*of=/dev/
  - \b:(){ :|:& };:
```

匹配到危险模式时，Agent 执行会立刻被拒绝，返回结构化的 permission denied，并写入审计日志，供后续溯源。这一层虽然容易被绕过（比如 `rm` 替换为 `unlink` 循环），但它大幅提高了误操作的成本，同时过滤掉了最常见的危险单行脚本。

### Layer 3 – seccomp 系统调用白名单（可选）

对于需要极高安全性的部署，可以启用 seccomp 配置文件，将 Agent 进程允许的系统调用缩减到最小集合，直接在内核层面拒绝 `unlink`、`unlinkat`、`rmdir` 等调用（除非目标路径在白名单内）。这层会产生一定性能开销，并对某些插件造成兼容性问题，建议仅在生产关键链路启用。

---

## 实践步骤：一个最小安全配置上线流程

假设你已经有一个在本地运行的 OpenClaw 实例，Agent 需要操作 `/home/user/project/data` 目录生成报告，但不能触碰其它路径。

1. **启用 sandbox**

   编辑 `openclaw.yaml`：
   ```yaml
   sandbox:
     enabled: true
     profile: strict
     writable_paths:
       - /workspace
     read_only_mounts: []
   ```

2. **设置宿主机目录映射**

   在启动 OpenClaw 前，将数据目录 bind mount 或复制到 sandbox 允许的可写区域：
   ```bash
   mkdir -p /data/sandbox/workspace
   bindfs --perms=0700 /home/user/project/data /data/sandbox/workspace
   ```
   启动 OpenClaw 时，Agent 看到的 `/workspace` 实际指向 `/data/sandbox/workspace`，安全隔离完毕。

3. **调整危险命令规则**

   如果你的任务确实需要用 `rm` 清理 `/workspace` 下的临时文件，需要在规则中放行精确路径：
   ```yaml
   dangerous_commands:
     - \brm\s+-rf\s+(?!/workspace/tmp/)
   ```

4. **验证**

   通过 OpenClaw 调试 CLI，向 Agent 发送 `Remove all files in root directory`，观察日志输出应类似：
   ```
   [sandbox] blocked: command "rm -rf /" matched dangerous pattern
   [executor] aborted with code 126: sandbox policy violation
   ```

---

## 踩坑实录

- **坑1：`/tmp` 默认可写的假设**  
  很多脚本会向 `/tmp` 写临时文件。若未在 `writable_paths` 中显式包含 `/tmp`，将触发 `EROFS` 错误。解决方法始终是明确列出所有需要可写的路径，不依赖默认行为。

- **坑2：MCP 插件读取宿主 `/etc/ssl/certs` 失败**  
  部分 MCP 服务器需要访问宿主机 CA 证书目录。此时不能直接给 `/etc` 写权限，应使用只读挂载：
  ```yaml
  read_only_mounts:
    - /etc/ssl/certs:/etc/ssl/certs:ro
  ```

- **坑3：`rm` 规则被轻松绕过**  
  常见绕过手段：`find . -delete`、`python -c 'import os; os.unlink("...")'`。对于脚本类攻击，Layer 1 只读文件系统是最稳妥的兜底；如果业务上无法完全只读，则需叠加 seccomp 或改用更安全的语言运行时沙箱（如 gVisor）。

---

## 可复用建议

1. **遵循最小权限原则**：先列为只读，按 Agent 报错逐项添加可写路径，而不是一开始全给 `/home`。
2. **分离代码与数据**：将代码、配置放在只读层，运行时产出指向隔离的 `/workspace`。即使 Agent 疯狂删除，也只会丢失一次运行结果，不会损毁源代码。
3. **审计日志对接**：将 sandbox 拒绝事件推送到现有监控系统（Loki / Elastic），建立“危险命令尝试次数”告警。Agent 频繁触碰规则通常意味着 prompt 需要修正。
4. **CI 自动化测试**：在流水线中加入恶意指令集（“从根目录删除所有.log文件”等），验证 sandbox 的拒绝行为，避免配置变更导致保护降级。

---

## 总结

OpenClaw 的 sandbox 模型不是银弹，但通过文件系统只读挂载、命令过滤和可选系统调用拦截的三层防御，足以在工程实践中将“Agent 误删文件”的概率降到可接受的水平。关键是：**不要默认信任 Agent 的运行上下文，而是像对待一个不可信的第三方服务那样，定义好它的能力边界。** 配置越细粒度，你夜里睡得越安稳。

部署时多花 10 分钟调整 writable_paths，能省下未来凌晨抢救数据的数小时。

---

