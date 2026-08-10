---
title: OpenClaw 沙盒安全模型深度解析：为什么你的 Agent 不会误删文件
feedId: 32358
source: 综合讨论
publishedAt: 2026-08-10
---

## 背景：当 Agent 拥有文件写权限时

在构建 OpenClaw Agent 处理本地任务时，我们不可避免要授予其读写文件的权限——整理下载目录、重命名批量文件、生成报告并归档。权限一旦开放，焦虑随之而来：一次 prompt 歧义、一个工具调用链路异常，就可能让 Agent 把重要目录当成临时缓存给清理掉。MCP 工具集和自定义插件越丰富，潜在风险面越大。

社区中常见的安全讨论往往走向两个极端：要么完全禁用写操作（导致大量自动化场景不可用），要么在宿主机直接裸奔，靠“相信 AI 的判断”祈祷不出事。OpenClaw 的沙盒安全模型提供了第三条路——可配置的、多层防御的文件系统隔离，让 Agent 在受限环境中“想删也删不掉”。

## 问题：一次险些发生的事故

先看一个真实复现场景：用户让 Agent “清理当前项目下超过 7 天的临时构建产物”。由于 prompt 未明确限定目录，Agent 通过 Shell 工具执行：

```bash
find . -type f -name "*.tmp" -mtime +7 -exec rm {} \;
```

如果当时工作目录恰好在用户根目录，这个命令会递归扫描整个家目录，误删其他项目的缓存文件甚至配置文件。在无隔离环境下，这就是一次典型的小指令引发大故障。

## 核心做法：三层沙盒模型

OpenClaw 的沙盒安全并非单一开关，而是由三层可独立配置的机制叠加而成，每一层都能单独截断风险操作。

### 第一层：文件系统视图隔离（Filesystem Root Jail）

最直接的防护手段是为每个 Agent 会话指定一个独立的文件系统根（`sandbox.root`）。Agent 及其调用的工具进程只能看到该目录子树，任何绝对路径（如 `/etc/passwd`）或 `../` 逃逸都会被内核级别的 `chroot` 或容器化实现拦截。

配置示例（`agent.yaml`）：
```yaml
sandbox:
  root: /home/user/agent-workspaces/project-a
  writable: true
```

此时 Agent 执行 `rm -rf /` 实际作用域是 `/home/user/agent-workspaces/project-a`，宿主机及其他用户目录不可见。这一层解决了“超出预期作用域”的问题，但对于限定目录内的误删仍无能为力。

### 第二层：操作白名单与模式拦截

第二层通过规则引擎（通常用 OpenClaw 的 `policy` 配置）定义“可以做什么”，而非“不能做什么”。典型策略：

- 禁止递归删除（拦截 `rm -r` 或 `find ... -exec rm`）
- 禁止修改隐藏文件（以 `.` 开头的文件）
- 禁止操作特定后缀（`.key`, `.pem`, `.env`）
- 写入前强制要求显式确认（`require_explicit_approval: true`）

策略示例：
```yaml
policies:
  - name: block-recursive-rm
    pattern: "rm\\s+-r"
    action: deny
    message: "递归删除被策略拦截，请明确指定文件列表"
  - name: protect-dotfiles
    path_pattern: "/\\..*"
    operations: [delete, rename]
    action: deny
```

这层可以做得很细，但维护成本随之升高。实践中建议先开启“危险操作拦截”预置规则集，再根据任务类型逐步放松。

### 第三层：写时复制与快照回滚

最高级的安全保障是让“删除”变为可逆操作。OpenClaw 支持对接 copy-on-write 文件系统（如 btrfs、ZFS）或 overlayfs，在 Agent 会话启动前创建快照。所有写操作都在差异层进行，会话结束后可以选择保留或丢弃变更。即使 Agent 清空了整个工作目录，只需一次回滚即可恢复到会话前状态。

Docker 驱动下的典型启动参数：
```bash
docker run --rm \
  -v /data/agent-sandbox/base:/base:ro \
  -v /data/agent-sandbox/upper:/upper \
  -v /data/agent-sandbox/work:/work \
  --storage-opt size=10G \
  openclaw-agent --sandbox.mode=overlay
```

这样三层叠加：路径 jail 限制范围 → 策略白名单阻止危险调用 → CoW 快照兜底，形成纵深防御。单一层的失效不会直接导致宿主机数据丢失。

## 踩坑点与工程教训

在实践中，以下问题反复出现：

1. **Jail 路径与工具预期路径不一致**  
   很多 MCP 工具或脚本会硬编码 `/tmp`、`/home/user` 等路径。将 root 设为 `/workspace` 后，工具会因找不到路径直接报错退出。解决方式是提前检查工具代码，或使用 `mount --bind` 将宿主机的 `/tmp` 映射进 jail（只读）。

2. **策略粒度太粗导致 Agent 无法工作**  
   比如为了安全拦截所有 `rm` 命令，结果 Agent 连正常的临时文件清理都做不了，任务卡死。建议分层策略：核心目录（如源代码）严格保护，临时输出目录（如 `build/`）允许删除，但要限定范围不能递归到上层。

3. **CoW 层磁盘膨胀**  
   长时间运行的 Agent 会话如果频繁写大量文件，差异层可能迅速占满磁盘。务必设置 `--storage-opt size=...` 并监控 overlay 使用率。定期提交或丢弃会话变更。

4. **权限与用户混淆**  
   在 Docker 中，Agent 进程默认以 root 运行，即使有 jail 也容易造成内部文件属主混乱。最好以非特权用户运行，并将工作目录 `chown` 给该用户。

## 可复用的最小安全配置建议

对于大多数自动化任务，一个开箱即用的“安全基线”可以这样组合：

- 使用 OverlayFS 的快照式沙盒（Docker 或 Podman 驱动）
- 开启默认拒绝策略（只允许白名单操作）
- 工作区独立，不与个人文件混布
- 风险命令（`rm -rf`、`mv` 跨目录）触发人工确认
- 会话日志完整记录所有 shell 调用，便于事后审计

如果资源有限，至少做到：会话 root 限定 + 禁止递归删除 + 快照（或定时备份）。这个组合足以防止绝大多数误删事故。

## 总结

Agent 误删文件不是“是否会发生”的问题，而是“何时发生以及爆炸半径多大”的问题。OpenClaw 的 sandbox 模型本质是用工程手段收敛爆炸半径：让 Agent 在一个受限、可观测、可回滚的环境中执行破坏性操作。三层机制分离了“范围控制”“行为控制”“后果控制”，每一层都能独立生效，组合后产生强安全保证。不需要因噎废食禁用文件操作，也不必在焦虑中裸奔——配置合理的安全策略后，你可以放心地把文件整理这类工作交给 Agent，而它即使“想使坏”，也翻不出沙盒的边界。

---

