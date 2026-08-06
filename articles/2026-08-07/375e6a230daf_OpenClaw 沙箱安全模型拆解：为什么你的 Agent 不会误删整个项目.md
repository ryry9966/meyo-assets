---
title: OpenClaw 沙箱安全模型拆解：为什么你的 Agent 不会误删整个项目
feedId: 31916
source: 综合讨论
publishedAt: 2026-08-07
---

## 背景：当 Agent 拿到文件系统权限

给 Agent 开放文件读写能力，是自动化开发场景中最常见的一步。一个典型的 OpenClaw 工作流可能是这样：你通过 MCP 工具或自定义插件，让 Agent 读取配置文件、写入日志、生成代码文件、清理临时目录。这些操作本质上是直接调用了宿主环境的文件系统接口——在 Node.js 里是 `fs` 模块，在 Python 里是 `os` / `shutil`。

问题也随之而来：Agent 并不真的“理解”它在操作什么。一个 Prompt 里的歧义，一个模型幻觉，就有可能让 `delete_temp_files()` 被错误调用在项目根目录，而不是 `/tmp/run-xxxx/` 下。缺少约束的话，这就是一次生产事故。

## 问题拆解：为什么传统的权限控制不够

你可能首先想到的是操作系统级别的用户隔离，比如用 `chroot`、Docker 容器、或者挂载只读文件系统。这些方案确实有效，但会带来几个工程上的摩擦：

1. **粒度太粗**：大多数隔离方案只能限定某个目录，无法表达“允许写入 `./output/`，但不允许删除 `./output/archive/`”这种细粒度策略。
2. **与工具逻辑脱节**：Agent 调用某个 MCP 工具时，工具内部可能会多次访问文件系统，操作系统只能看到进程级别的系统调用，难以区分“这是 Agent 的意图”还是“本来合法的内部操作”。
3. **调试成本高**：一旦 Agent 被意料之外的权限拒绝，很难立刻判断是策略问题还是模型输出问题，排查链路长。

OpenClaw 的沙箱模型在这层之上提供了一种**工具运行时级别的虚拟化**，它在文件系统调用和实际磁盘操作之间插入了一层代理，由沙箱统一收紧访问边界。

## OpenClaw 沙箱的运作方式

在 OpenClaw 中，Agent 并不会直接调用 Node.js 的 `fs.writeFileSync`，而是通过系统提供的 `SandboxFileSystem` 实例来完成所有 I/O。这个实例内部维护了一张**操作策略表**，大致长这样：

```
{
  "/workspace": { read: true, write: true, delete: false },
  "/workspace/output": { read: true, write: true, delete: true },
  "/workspace/config": { read: true, write: false, delete: false },
  "/workspace/node_modules": { read: true, write: false, delete: false }
}
```

当 Agent 试图写入 `/workspace/config/app.yml` 时，沙箱会检查最匹配的路径规则。如果策略表里 `config` 目录的 `write` 为 `false`，这次调用会直接被拒绝，并返回一个可控的沙箱错误，而不是让异常直接落在文件系统层。

更重要的是**路径规范化与穿越防御**。沙箱要求所有路径都必须解析为绝对路径，并过滤掉 `..`、符号链接跳转等绕过手段。即便 Agent 的输出里构造了 `../../etc/passwd`，沙箱在匹配规则之前会先将其规范化到 `/etc/passwd`，然后发现该路径不在已授权的挂载点内，直接拒绝访问。

### 为什么“误删文件”被天然阻断

误删通常来自两类场景：一是 Agent 生成了一条危险命令（比如 `rm -rf /workspace/src`），二是某个自动化工具内部逻辑异常，传入了一个未经验证的路径。

在 OpenClaw 沙箱下：

- **删除操作被显式建模**。`SandboxFileSystem` 没有提供一个通用的“执行 shell 命令”入口，文件删除必须通过 `remove(path)` 方法，而该方法会先检查 `delete` 策略。Agent 无法绕过这一层调用 `exec('rm -rf ...')`，除非你额外暴露了 Shell 工具并且没有约束它——这已经属于配置问题，不是沙箱本身的责任。
- **通配符与遍历被阻断**。沙箱不支持 glob 形式的批量删除，所有路径都必须是逐个校验的文本。如果 Agent 想删除 `output/*`，它需要先通过 `readdir` 列出文件，再逐一调用 `remove`，每一步都会接受策略检查。这极大地降低了批量误删的概率。
- **关键目录默认不可写不可删**。OpenClaw 的默认沙箱模板会把当前工作目录的 `.git`、`node_modules`、`package.json` 等关键文件标记为只读，同时禁止在这些路径上执行 `delete`。即便你手动给 Agent 写了一个“清理项目依赖”的任务，沙箱也会因为触碰了策略而立即拦截，并返回清晰的拒绝理由。

## 操作步骤：如何正确配置一个项目沙箱

在 OpenClaw 项目中，沙箱配置通常内嵌在 `openclaw.config.json` 或 `.openclaw/workspace.json` 中。一个安全的开发沙箱配置示例：

```json
{
  "sandbox": {
    "enabled": true,
    "mounts": [
      {
        "hostPath": "./src",
        "sandboxPath": "/workspace/src",
        "permissions": ["read", "write"]
      },
      {
        "hostPath": "./tests",
        "sandboxPath": "/workspace/tests",
        "permissions": ["read", "write"]
      },
      {
        "hostPath": "./output",
        "sandboxPath": "/workspace/output",
        "permissions": ["read", "write", "delete"]
      },
      {
        "hostPath": "./config",
        "sandboxPath": "/workspace/config",
        "permissions": ["read"]
      }
    ],
    "forbiddenOperations": ["exec", "spawn"],
    "denyPatterns": ["\\.env$", "\\.git/"]
  }
}
```

这里有几个关键点：

1. **最小权限原则**：只挂载 Agent 任务真正需要的目录，而不是整个项目根。
2. **分离读写删权限**：`output` 目录开放 `delete`，因为任务可能需要清理历史文件；而 `config` 只有 `read`，防止配置被意外改写。
3. **禁止 Shell 执行**：`forbiddenOperations` 里禁用 `exec` 和 `spawn`，防止 Agent 绕过文件沙箱直接调系统命令。
4. **拒绝关键文件**：`denyPatterns` 用正则加固，即便路径在挂载范围内，`.env` 和 `.git` 下的操作也会被额外拦截。

## 踩坑实录

实践中比较容易踩的几个点：

- **相对路径的坑**。开发者习惯在任务里写 `./output/report.md`，但沙箱只接受以挂载点为根的绝对路径。如果 Agent 没有拼接 `sandboxPath` 的能力，工具里需要在调用前自己做一层 `path.resolve(workspaceRoot, relativePath)` 的转换，否则会命中“路径不在沙箱内”的错误。你在调试时看到的提示往往是 `EACCES` 或 `SANDBOX_DENIED`，但很难立刻定位是路径问题。
- **挂载点覆盖冲突**。如果你同时挂载了 `/workspace` 和 `/workspace/src`，沙箱会优先匹配最长前缀的规则。如果你给 `/workspace` 写了 `delete:false`，而 `/workspace/src` 没配置，则 `/workspace/src` 会继承父级的 `delete:false`。建议显式声明每个子目录的权限，避免继承带来的意外拒绝。
- **符号链接逃逸**。沙箱默认禁止跟随宿主的符号链接，但如果你在挂载目录内创建了一个指向外部文件的软链接，且沙箱没有关闭 `followSymlinks`，Agent 仍然可能通过这个链接读写外部文件。务必在生产配置里设置 `"followSymlinks": false`。
- **性能开销**。路径规范化、策略匹配、日志记录会略微拖慢 I/O，尤其在大量小文件读写时。可以考虑把高频读写的目录排除出沙箱（如果业务允许），或者在调试阶段先全开沙箱，正式环境再打开策略，避免影响核心流程。

## 可复用的工程建议

如果你正在为自己的 Agent 或 MCP 工具构建安全策略，以下几点可以快速复用：

1. **分层策略**：将文件操作分为三个信任等级——`trusted`（只读配置）、`scratch`（可读写可删除的临时区）、`output`（最终产物可写不可删）。让 Agent 的任务函数只接受这三种参数，而不是任意路径。
2. **日志与审计**：在沙箱层记录每一次 `delete`、`write` 操作的调用栈和 Agent 输入上下文。OpenClaw 支持将拒绝事件发送到指定的 audit hook，一旦出现异常拒绝，你可以回溯到具体哪个 Prompt 触发了危险操作。
3. **测试里故意使坏**：写 E2E 测试时，故意给 Agent 发送“删除 src 目录”的指令，看沙箱是否正确拦截，并确认 Agent 能根据拒绝信息调整行为，而不是卡死或给出错误回复。这种测试能有效防止未来 Prompt 调整导致的安全退化。

## 总结

OpenClaw 的沙箱并不是一个黑盒魔法，而是一套在文件系统调用层面强制执行的策略引擎。它之所以能防止 Agent 误删文件，是因为它将“删除”这种危险操作从 Agent 的隐含能力中剥离出来，变成了一个需要显式授权、逐条核验的动作。配合合理的挂载权限、禁止 Shell 执行和拒绝关键文件的正则，你可以在几乎不影响开发效率的前提下，把文件操作风险降到可控范围。

安全这件事，不能靠“我希望模型别犯错”，而要靠工程手段让错误无法发生。

---

