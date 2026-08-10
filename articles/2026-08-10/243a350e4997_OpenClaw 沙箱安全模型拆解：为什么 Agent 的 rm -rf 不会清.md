---
title: OpenClaw 沙箱安全模型拆解：为什么 Agent 的 rm -rf 不会清空你的 home 目录
feedId: 32364
source: 综合讨论
publishedAt: 2026-08-10
---

# OpenClaw 沙箱安全模型拆解：为什么 Agent 的 rm -rf 不会清空你的 home 目录

## 背景：当 Agent 拿到 shell 的瞬间
很多跑过自动化 Agent 的同学都有过这样一个瞬间：你给 Agent 配置了一个 MCP 服务器，挂上了文件读写工具，随手给了个系统指令——“把我 Desktop 上的 tmp 文件夹清一下”。然后你盯着终端，咖啡都忘了喝。

传统机器人或脚本自己写还好，但现在是 LLM 驱动的 Agent，它可能理解偏差，把 `Desktop/tmp` 理解成 `~/Desktop`，或者产生幻觉直接执行 `rm -rf ~/Desktop`。一旦发生，就是事故现场。

OpenClaw 的解决思路是：**默认不信任 Agent 的执行环境**，通过内置的 sandbox 安全模型把危险操作限定在一个隔离域内。这篇文章从工程实现角度拆一下这套模型怎么工作，以及在真实项目里怎么配才安全、怎么踩坑。

## 问题拆解：Agent 误删的几个人祸高发点
先对齐几个典型风险场景：

1. **命令误解**：Agent 把 “清理临时文件” 理解为 “删掉所有 .tmp 文件”，然后全盘扫描删除。
2. **路径混淆**：Agent 把相对路径和绝对路径搞混，在 `/home/user` 下执行了 `rm -rf *`。
3. **工具越权**：MCP 文件工具直接暴露了整个文件系统的读写能力，没有限制作用域。
4. **状态残留**：Agent 在多次对话中携带的上下文里有个 “临时目录” 变量，结果指向了系统关键路径。

这些问题本质上不属于 AI 能力问题，而是**权限设计问题**。所以安全方案必须落在系统层，而不是靠 prompt 约束。

## OpenClaw 的 sandbox 是怎么设计的（做法/步骤）
OpenClaw 的文件操作 sandbox 并不是一个简单的 `chroot`，而是结合了**文件系统视图隔离 + 操作审计 + 回滚能力**的三层结构。下面按配置步骤来说。

### 1. 声明工作区（workspace）
在 OpenClaw 的项目配置里，每个 Agent 实例都会绑定一个 `workspace` 目录，这是它的根路径（`/`）。例如：

```yaml
sandbox:
  workspace: /home/user/openclaw/agent-workspaces/agent-1
  readonlyRootfs: true
```

Agent 看到的文件系统树是：
```
/  
├── task/    （可读写，映射到 workspace）
└── data/    （可选挂载）
```
它看不到 `/etc`、`/home`，甚至不能 `cd ..` 越狱，因为文件系统视图已经在进程级被截断了。实现上用的是 Linux 的 `mount namespace` + `pivot_root` 或 `overlayfs`，而非简单的软链接限制。

### 2. 限制文件操作工具的白名单路径
MCP 服务器的文件工具（read/write/edit/delete）会被强制注入一个 `allowedPaths` 数组，只允许在 `workspace` 下操作。例如在给 Agent 提供工具的代码里会做：

```typescript
function resolvePath(requestedPath: string, workspace: string): string {
  const resolved = path.resolve(workspace, requestedPath);
  if (!resolved.startsWith(workspace)) {
    throw new Error("Access denied: path traversal detected");
  }
  return resolved;
}
```

即使 Agent 在对话里说“请删除 `/etc/passwd`”，最终调用的工具会直接抛 `Access denied`。**权限校验在工具实现侧，而非 LLM 侧**，这是关键。

### 3. 只读根文件系统 + 写时复制
开启 `readonlyRootfs: true` 后，根文件系统被 `mount --bind -o ro` 挂载成只读。Agent 执行的任何写入都被定向到 overlay 的上层可写层，而这个可写层是一个临时目录，任务结束后会被丢弃或按需提交。这意味着：

- 脚本里写 `rm -rf /usr` 会直接报错 `Read-only file system`。
- 真正的宿主机文件不受任何影响。

### 4. 审计日志 + 回滚
每次沙箱内的文件修改都会被记录在最底层 fs 的 `upperdir` diff 里。如果 Agent 在 workspace 内误删了自己的产出文件，可以用 `sandbox diff` 命令回滚一步或多步。这个 diff 不是 git 仓库，而是文件系统层的快照，更适合自动化场景下的文件恢复。

## 踩坑实录：你以为安全了，其实没有
在实践中，这三个点翻车最多：

### 坑1：挂载了 home 目录还不设 readonly
为了 “让 Agent 访问我的常用数据”，有人直接把 `~/` 挂载到了 `/data`，但没有开 `readonly`。结果 Agent 在 `/data` 下执行清理，导致原目录被删。**正确做法是：数据只读挂载，输出只写到 workspace**。要交换数据，使用专用的 `inbox` / `outbox` 模式。

### 坑2：通过环境变量泄露出真实路径
Agent 可能通过 `printenv` 拿到宿主机的 `HOME`、`PWD` 等环境变量，然后尝试用其他不受限的工具（比如 shell 脚本里 curl 上传本地文件）间接操作。现在 OpenClaw 会默认清空或伪造环境变量，比如把 `HOME` 设成 `/workspace`，PWD 设为 `/workspace/task`。但老版本需要手动在启动配置中显式设置 `clearEnv: true`。

### 坑3：第三方 MCP 服务没做路径限制
自己写的工具模块有 `allowedPaths` 保护，但如果你接了一个社区 MCP 服务器，它可能未实现路径白名单。所以在引入外部工具时必须检查其代码，或者在 OpenClaw 侧通过中间件对工具请求做二次过滤。

## 可复用建议：一个生产级安全配置基线
不管你是跑代码生成、文档整理还是自动化测试，建议把这套基线写进项目配置模板：

```yaml
sandbox:
  mode: strict
  workspace: /srv/agent-sessions/{{.SessionID}}
  readonlyRootfs: true
  mountpoints:
    - src: /data/shared/inbox
      dst: /inbox
      mode: ro
    - src: /data/shared/outbox
      dst: /outbox
      mode: rw
  env:
    clear: true
    overrides:
      HOME: /workspace
      PWD: /workspace/task
  timeout: 600s
  maxDisk: 1GB
```

额外三条日常操作纪律：
- **永远不要** 把 `workspace` 路径指向真实项目源码目录，用 `rsync` 或 `git clone` 把需要的数据复制进去。
- **每次新版 Agent 上线前**，在 sandbox 内用模糊测试脚本（比如随机删除、写满磁盘）跑一轮冒烟。
- **打开审计日志的持久化**，方便事后定位是不是沙箱自身的行为，还是中间件被绕过。

## 总结
OpenClaw 的 sandbox 安全模型本质是 “默认不信任” 的设计哲学。它用 mount namespace 隔离文件视图、在工具层做强路径校验、把根系统变成只读并叠加写时复制，最后加上审计回滚，形成了一套多层防御。

对于自动化实践来说，这意味着你再也不用在深夜下意识地检查 Agent 的清理命令是不是多打了一个 `/`。沙箱会替你把所有越界操作拦在墙外，你只需要专注把数据交换的边界配清楚就好。

说到底，Agent 不会误删文件的秘密，不是它变聪明了，而是它根本没被给到那把钥匙。

---

