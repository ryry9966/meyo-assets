---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删你的文件
feedId: 31102
source: 综合讨论
publishedAt: 2026-07-31
---

## 背景：当 "rm -rf" 遇到自动化 Agent

在 Agent + 命令执行 (Shell/Terminal MCP) 的实践中，最常见也最让人后背一凉的场景是：AI 自动生成了一条清理命令，结果路径拼错、变量为空，瞬间把宿主机的关键目录删掉。传统的解决办法是依赖权限（避免 root）、用 Docker 做容器隔离，或者靠人工审查每一次操作。但这些方案要么侵入性太强，要么影响开发体验——我们在本地调试文件、跑脚本时，很难忍受每次操作都去确认。

OpenClaw 从设计之初就内置了一个基于 Linux 命名空间的沙箱（sandbox）安全模型。它不依赖 Docker，而是利用 `bubblewrap` 及相关内核特性，构造轻量、无感知的隔离执行环境。这篇文章拆解它的工作机制，以及你在插件和自动化场景下需要注意的配置细节。

## 问题拆解：Agent 真的能做到“删不掉”吗？

要让 Agent 无法误删文件，不能仅靠提示词约束。正确的思路是在执行层做**文件系统视图隔离** + **写入重定向**。具体来说，OpenClaw sandbox 解决了三个问题：

1. 宿主重要目录对 Agent 进程不可见或只读；
2. Agent 的所有写操作被限制在一个可丢弃的层中，不污染原始文件；
3. 即使 Agent 尝试非法系统调用（如直接调用 `unlink` 非授权路径），也会被 seccomp 规则拦截。

这意味着，即便远程模型返回了 `rm -rf /` 这样的极端指令，执行结果也只是在沙箱内部失败，宿主文件毫发无损。

## 底层做法：bubblewrap + 写入叠加层

OpenClaw 在启动 Agent 会话时会解析 `.claw.yaml` 中的 `sandbox` 配置段，生成如下结构的隔离环境（以 Linux 为例）：

- 使用 `bwrap` 创建新的 mount namespace 和 user namespace；
- 宿主 rootfs 以 `ro-bind`（只读绑定挂载）方式挂载到沙箱内的 `/`；
- 用户指定的工作目录（如 `./workspace`）通过 `bind` 挂载为可写，或者通过 `overlayfs` 创建 tmpfs 上层，实现“看起来可读写，实际上不会回写原始目录”的效果；
- `/proc`、`/dev` 等伪文件系统按需挂载，避免 agent 获取宿主机进程信息；
- seccomp 规则默认放行常用系统调用，但屏蔽 `mount`、`pivot_root` 等可逃逸沙箱的高危操作。

核心命令片段（简化版）：

```bash
bwrap \
  --ro-bind / / \
  --bind ./workspace /home/agent/workspace \
  --tmpfs /tmp \
  --proc /proc \
  --dev /dev \
  --unshare-user --unshare-ipc --unshare-pid \
  --seccomp 9 \
  -- agent-cli run
```

当 Agent 在沙箱内执行 `ls /etc`，它实际看到的是宿主机 `/etc` 的只读副本；执行 `echo > /etc/test` 会因只读文件系统而失败；执行 `rm /home/agent/workspace/important.log` 则作用于写入层，不会影响宿主机对应文件（如果采用 overlay 且没有 `--bind` 原始目录）。

## 配置与复现步骤

### 1. 环境准备
确保系统内核 ≥ 4.19，已安装 `bubblewrap`（`apt install bubblewrap` 或 `brew install bubblewrap`）。OpenClaw 会自动检测沙箱所需依赖，缺依赖会降级为 warning 模式，但安全边界变弱。

### 2. 沙箱配置
在项目根目录的 `.claw.yaml` 中添加：

```yaml
sandbox:
  enabled: true
  writable:
    - ./data          # 允许写的工作目录
    - /tmp
  readonly_host:
    - /usr/share
    - /etc/ssl
  network: false      # 无网络需求时可关闭
  seccomp: default
```

– `writable` 列表必须显式声明，Agent 无法创建或删除列表之外的文件；  
– 如果需要访问 `/etc/resolv.conf`、SSL 证书等，务必加入 `readonly_host`，否则 DNS 和 HTTPS 请求会失败；  
– `network: false` 会额外创建 network namespace，阻断所有外连。

### 3. 启动并验证
启动 Agent 处理一个带有文件删除意图的任务，例如“清理 workspace 下的临时文件”。用 `strace` 或日志确认沙箱内的操作不会穿透到宿主。你还可以故意让 Agent 执行 `rm -rf /usr/bin`，观察命令因只读挂载报错，宿主机目录完好。

## 踩坑点：这些情况仍需显式处理

1. **/tmp 与临时文件依赖**  
   很多工具（git, ffmpeg, pip）默认向 `/tmp` 写临时文件。若未将 `/tmp` 加入 `writable`，这些命令会直接崩溃。建议总是将 `/tmp` 挂载为 tmpfs，性能好且自动清理。

2. **叠加层的空间回收**  
   如果使用 overlay 且 Agent 产生大量写操作，临时层会不断变大。长时间运行的任务需设置磁盘水位告警，或者定时重建沙箱。

3. **文件所有权 uid/gid 映射冲突**  
   沙箱内 Agent 运行的 uid 可能是 1000 或 999，而宿主机同名 uid 的文件权限可能不一致。最佳实践是使用 `--unshare-user` 做用户映射，或在 `writable` 目录下预先设置 `chmod -R 777`（仅限非敏感数据）。

4. **seccomp 切断特殊调用**  
   某些 MCP 插件会依赖 `ptrace`、`perf_event_open` 等系统调用。在默认 seccomp 规则下会被拦截。此时不要关闭 seccomp，而是用 `seccomp: custom-profile` 单点放行。

## 可复用建议

- **永远使用非 root 身份启动 OpenClaw**。即使沙箱逃逸，普通用户权限也能把损害降到最低。
- **`writable` 目录尽量用独立挂载点**，不要将 Git 仓库根目录直接设为可写，而是用 `./workspace/staging` 这样的子目录。
- **开启沙箱日志审计**，`sandbox.log: true` 能把每次文件修改记录到 journal，方便回溯。
- **配合只读挂载的策略**：把 `/home/user/.ssh`、`/boot` 等敏感路径显式列为只读，可以杜绝信息泄漏。
- **多阶段任务尽量复用同一 sandbox 实例**，避免频繁创建销毁导致 tmpfs 碎片化。

## 总结

OpenClaw 的 sandbox 安全模型不是魔法，而是将 Agent 的执行环境严格限定在只读宿主机视图 + 可控写入层的工程方案。它让开发者可以放心地把文件系统的一部分暴露给 AI，而无需担心一次错误的模型输出就导致灾难性丢失。即使你是非安全背景的自动化实践者，只要遵循“最小可写”原则配置 `.claw.yaml`，并在初次部署时花半小时走一遍上面的验证步骤，就能在日常工作中实现“Agent 不会误删文件”这个朴素却关键的安全目标。

---

