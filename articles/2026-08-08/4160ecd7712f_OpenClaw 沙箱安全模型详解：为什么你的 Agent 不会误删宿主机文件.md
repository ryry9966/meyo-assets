---
title: OpenClaw 沙箱安全模型详解：为什么你的 Agent 不会误删宿主机文件
feedId: 32071
source: 综合讨论
publishedAt: 2026-08-08
---

# OpenClaw 沙箱安全模型详解：为什么你的 Agent 不会误删宿主机文件

## 背景：每一次 rm -rf 的噩梦

用 Agent 做自动化的时候，最怕什么？不是任务失败，不是 API 调用超时，而是它拿到 shell 权限之后，一个删除命令把你本机项目目录清空了。这种担忧在社区里不算杞人忧天——哪怕只是跑一个「帮我整理下载文件夹」的任务，如果工具允许直接调用 `os.remove()` 或者 `shutil.rmtree()`，翻车的代价就是无法挽回的数据丢失。

很多 MCP Server 和插件在设计时，为了图省事，直接暴露了宿主文件系统的读写接口。这相当于给 Agent 开了一张不设限制的 Sudo 票。一旦 prompt 里路径写错，或者 LLM 幻觉出某个危险操作，后果就是灾难性级别。

OpenClaw 从运行时架构上就针对这个问题做了隔离设计。它不是事后补救，而是默认假设 Agent 不可信——你不需要在 prompt 里反复强调「不要删文件」，因为 Agent 根本就没有权限碰到关键路径。

下面我们来看看这套沙箱模型到底做了什么，以及实际使用中你要注意哪些配置细节。

## 核心问题：文件访问可以被限制到什么粒度？

常见的文件安全控制有两种思路：

1. **事后校验**：在 Agent 执行写操作前拦截，用规则或模型判断是否允许。这依赖 prompt 上下文，也容易被绕过。
2. **能力剥夺**：从操作系统/容器层面直接禁止访问，Agent 连感知都感知不到。

OpenClaw 选用的是第二种，也就是强制沙箱（Sandbox）。Agent 在执行任务时运行在一个受控的文件系统视图里，这个视图由运营者（你）显式声明。你能精确控制：

- 哪些目录对 Agent 可见
- 哪些路径是只读，哪些可读写
- 是否允许访问 `/home`、`/etc`、根目录
- 是否允许执行某些系统调用（如 `chmod`、`chown`）

这个模型的设计目标很明确：**Agent 没有全局视图，就没有误删除全局文件的可能性。**

## 实现原理：三层隔离

OpenClaw 的沙箱不是简单的 chroot，而是通过文件系统 access proxy + seccomp + 用户命名空间的组合来保证隔离。具体来说分三层：

### 第 1 层：虚拟文件系统挂载（Access Proxy）

Agent 实际接触的根目录，是通过 FUSE 或 virtiofs 之类的方式挂载的一个「视图文件系统」。这个文件系统会依据你的项目配置，只嫁接出允许访问的那部分目录树。比如你只允许访问 `./workspace`，那 Agent 眼里就只有这一个目录，看不到 `/home/user` 甚至看不到 `/etc`。

这层的好处是：就算 Agent 执行 `rm -rf /`，它破坏的也只是你自己限定的 workspace 镜像，你的真实系统毫发无损。

实际配置时，在 OpenClaw 的沙箱声明文件（`sandbox.yaml` 或项目内 `claw.toml`）里，可以这样写：

```yaml
filesystem:
  mounts:
    - host_path: "./project_data"
      mount_point: "/workspace"
      access: "readwrite"
    - host_path: "./config"
      mount_point: "/config"
      access: "readonly"
```

上面示例中，Agent 只能看到 `/workspace` 和 `/config`，且后者只读。它连 `/tmp` 都不会有，除非你显式声明。

### 第 2 层：系统调用过滤（seccomp）

有些危险操作不需要文件访问，比如执行 `rm` 可能触发 `unlinkat` 系统调用。OpenClaw 的运行时内置了一套 seccomp BPF 规则，默认拦截：

- `unlink` / `unlinkat`（文件删除）
- `rename` / `renameat2`（重命名/覆盖）
- `chmod` / `chown`（权限修改）
- `mount` / `umount2`（挂载操作）

如果想要 Agent 拥有删除能力，你必须有意识地在规则中放行这些 syscall。默认情况下是全部拦截的。

### 第 3 层：用户与组隔离（User Namespace）

即使 Agent 真的逃逸出了文件系统视图，也还有一层兜底：它在沙箱里运行的是一个非特权用户，uid 和宿主机不重叠。这意味着就算它能访问到宿主机路径，也因为没有读写权限而无法造成破坏。这种 uid mapping 是容器沙箱的经典范式，OpenClaw 用 user namespace 实现了无 root 的运行环境。

## 踩坑点：沙箱开得太小导致任务失败

三层隔离虽然安全，但实际使用中不少同学会因为过度限制，导致 Agent 连正常的任务都无法完成。常见坑有：

**1. 忘记暴露临时目录**

很多工具（特别是 Python 脚本）会在 `/tmp` 下写临时文件。如果你没有在 mounts 中显式加上 `/tmp`，Agent 调用 `tempfile` 会直接报 Permission denied。解决办法是加一条：

```yaml
- host_path: "/tmp/claw-sandbox"
  mount_point: "/tmp"
  access: "readwrite"
```

并且宿主机上这个路径要真实存在。

**2. 对 npm/pip 等包管理器不友好**

Agent 如果想临时安装某个 Python 包来完成任务，会往系统路径写文件。这通常会被只读挂载拦截。建议的策略是：不要在 Agent 任务中执行动态安装，如果需要额外依赖，预先在 Docker 镜像或 sandbox 模板里装好。

**3. 只读路径下 Agent 尝试写文件导致循环重试**

如果你把一个目录设为只读，但 Agent 的任务是「修改配置文件并保存」，它可能会反复尝试写、失败、然后重试，消耗大量 token。经验是：任务设计时要明确可写路径，并在 prompt 里提示 Agent 使用 `/workspace` 作为输出目录。

## 可复用建议：按任务类型选择沙箱策略

结合社区实践，有三套配置模板可以快速套用：

- **纯数据分析任务**（读数据、出报告）：只挂载数据目录 readonly，输出写 `/workspace/reports`，不允许删除和权限修改。最严苛。
- **文件批处理任务**（重命名、格式转换）：允许 rename 系统调用，但限定操作在某个结构化目录下，如 `/workspace/inbox` → `/workspace/outbox`。
- **开发辅助任务**（修改代码、运行测试）：挂载整个项目目录 readwrite，但限制不能访问 `.git` 以外的点文件，防止 Agent 乱改配置。同时放行 `rm` 和改名，但要监督执行。

你也可以基于环境变量动态调整：对线上长期运行的 Agent 保持只读，对本地调试可以略放开。

## 总结

OpenClaw 的沙箱模型之所以能防住「Agent 误删文件」，是因为它不是靠提示词约束，而是靠**视角隔离 + 系统调用过滤 + 用户隔离**的结构化安全。这不只是一个功能，而是一种工程假设：把 Agent 当作不受信任的第三方程序，从 OS 层面收束它的能力。

当然，安全性和灵活性永远有 trade-off。沙箱越紧，Agent 能做的事就越少。实际配置时关键是提前想清楚任务需要的最小权限，然后只打开那部分，其他一律封锁。这样即使 LLM 产生幻觉，也会在 OS 调用那一层被硬阻断，你早上起来才不会发现自己的代码仓库被清回了初始化状态。

> 目前 OpenClaw 的 sandbox 已经支持在 Linux/macOS 上本地运行，也兼容远端 Docker 执行器。更多配置细节可查阅 OpenClaw 文档中的「Sandbox & Execution Profile」一节。

---

