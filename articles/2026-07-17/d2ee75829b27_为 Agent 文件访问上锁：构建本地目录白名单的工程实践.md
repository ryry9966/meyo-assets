---
title: 为 Agent 文件访问上锁：构建本地目录白名单的工程实践
feedId: 29404
source: 综合讨论
publishedAt: 2026-07-17
---

# 为 Agent 文件访问上锁：构建本地目录白名单的工程实践

## 一、为什么 Agent 需要一个文件访问护栏

当 Agent 能够自主执行命令、读写文件时，它的能力边界被大大拓宽了——可以自动整理你的下载文件夹、批量重命名图片、根据规则归档日志。但随之而来的是一个令人不安的问题：如果它“手滑”删除了 `~/.ssh` 或者是把 `/etc/hosts` 给覆盖了怎么办？

在 OpenClaw 这类工具里，Agent 通过 MCP Server 或自定义脚本执行动作，往往是以当前用户身份运行的，拥有该用户的所有文件访问权限。这就意味着一个错误的 prompt、一段未经验证的规划、甚至一个幻觉产生的路径，都可能造成无法挽回的数据损坏。最典型的场景是：你让 Agent “清理项目目录下的临时文件”，但它却把“项目目录”理解成了家目录，然后开始递归删除。

因此，为文件操作建立一道基础护栏——**本地目录白名单**，是让自动化真正可用的必要条件。它不是银弹，但能覆盖绝大多数意外破坏。

## 二、设计思路：在文件系统调用处设卡

我们的目标是：拦截 Agent 产生的所有文件写入/删除操作，只允许它们在预先声明的目录下进行。读取操作可以适当放宽，但为防止信息泄露（如 Agent 读到包含密钥的配置文件），也建议纳入管理。

在 Linux 环境下，最轻量的实现方式不是内核模块或容器，而是利用 **LD_PRELOAD** 劫持 libc 的文件操作函数，例如 `open`、`fopen`、`unlink`、`rename` 等。当 Agent 的进程（或它启动的子进程）尝试访问文件时，首先被我们的劫持库拦截，检查目标路径是否落在白名单内。白名单在初始化时从环境变量或配置文件中读取。

在 macOS 上，由于 SIP 保护，无法对系统进程使用 `DYLD_INSERT_LIBRARIES`，但我们可以对 Agent 自身及其直接 spawn 的子进程（例如通过 `execve` 包装，显式设置 `DYLD_INSERT_LIBRARIES`）进行保护，这会覆盖常见场景。

对于 OpenClaw 用户，如果你的 Agent 通过 shell 执行命令，那么实际上所有文件操作都来自子进程。你可以在 agent 入口脚本中 preload 这个库，对所有后代生效。

## 三、实现步骤：一个最小可行的 open/unlink 白名单守卫

以下是基于 C 语言的共享库实现，支持 `open`, `openat`, `unlink`, `unlinkat`, `rename`, `renameat` 等关键函数。只展示核心逻辑。

**白名单判断函数：**

```c
static int is_path_allowed(const char *path) {
    if (path == NULL) return 0;
    char resolved[PATH_MAX];
    if (realpath(path, resolved) == NULL) {
        // 文件不存在时，检查父目录是否在白名单内
        char *dir = dirname(strdupa(path)); // 简化处理
        if (realpath(dir, resolved) == NULL) return 0;
    }
    for (int i = 0; i < allowed_dirs_count; i++) {
        if (strncmp(resolved, allowed_dirs[i], strlen(allowed_dirs[i])) == 0) {
            // 确保匹配的是完整目录组件：要么完全相等，要么 resolved 的下一个字符是 '/'
            if (resolved[strlen(allowed_dirs[i])] == '\0' ||
                resolved[strlen(allowed_dirs[i])] == '/') {
                return 1;
            }
        }
    }
    return 0;
}
```

**对 open 的拦截：**

```c
int open(const char *pathname, int flags, ...) {
    // 获取原始函数指针
    int (*original_open)(const char *, int, ...) = dlsym(RTLD_NEXT, "open");
    mode_t mode = 0;
    if (flags & O_CREAT) {
        va_list arg;
        va_start(arg, flags);
        mode = va_arg(arg, mode_t);
        va_end(arg);
    }
    // 只限制写入和创建操作
    if (flags & (O_WRONLY | O_RDWR | O_CREAT | O_TRUNC)) {
        if (!is_path_allowed(pathname)) {
            errno = EACCES;
            return -1;
        }
    }
    return original_open(pathname, flags, mode);
}
```

`unlink` 和 `rename` 的处理类似，因为这两个操作完全应该被白名单限制。

**编译和使用：**

```bash
gcc -shared -fPIC -o libfsguard.so fsguard.c -ldl
```

在启动 Agent 时，设置环境变量：

```bash
export ALLOWED_DIRS="/home/user/projects/agent-sandbox:/tmp/agent-work"
export LD_PRELOAD=/path/to/libfsguard.so
# 对于 macOS：
export DYLD_INSERT_LIBRARIES=/path/to/libfsguard.so
export DYLD_FORCE_FLAT_NAMESPACE=1
```

然后在该 shell 中执行所有 Agent 相关命令，文件写入、删除都将被限制在上述两个目录之内。尝试删除 `~/important.doc` 会直接收到 `Permission denied`。

## 四、踩坑与加固建议

1. **相对路径与符号链接**  
   `realpath()` 会解析符号链接并得到绝对路径，这非常重要。否则，一个指向 `/etc` 的符号链接 `/tmp/agent-work/link_to_etc` 可能被利用绕过白名单。代价是在某些不存在文件创建时，`realpath` 会失败，此时必须退而求其次，检查父目录，需注意 TOCTOU 问题，但在本地 Agent 场景下可接受。

2. **移动文件操作的安全性**  
   `rename` 同时涉及源和目标路径，两者都必须通过白名单检查。否则可能通过将白名单外的重要文件移入白名单内，再通过后续操作泄露或覆盖。

3. **动态链接库的加载顺序**  
   `LD_PRELOAD` 劫持可能被其他库或环境覆盖，要确保这个库是第一个被加载的。如果 Agent 自身有反调试或完整性检查，可能会有兼容性问题，可以改为用 `seccomp` BPF 过滤器，但复杂度会大幅上升。

4. **性能影响**  
   每个文件系统调用都增加了一次 `realpath` 和字符串匹配，对于高频 I/O 场景会有可感知的性能下降。建议白名单目录数量控制在个位数，并用前缀匹配替代整路径比较。也可缓存已检查路径的结果。

5. **Windows 上的替代方案**  
   Windows 上没有 LD_PRELOAD 等价物。可以使用 Detours 库拦截 Win32 API `CreateFileW`、`DeleteFileW` 等，或者将 Agent 放在一个专用用户下，配合 ACL 权限控制。后者更简单稳妥。

## 五、可复用性总结

- **入口统一**：所有 Agent 操作经过同一个 shell wrapper，环境变量和 preload 配置集中管理。
- **白名单配置化**：由外部 JSON/YAML 控制，可动态更新，无需重新编译库。
- **日志告警**：拦截时通过 `syslog` 或 stderr 记录被阻止的路径和调用栈，便于审计和调试。
- **分层防护**：白名单是最后一道线，配合 prompt 层面的约束（如“你只能操作 /workspace 下的文件”）形成纵深防御。

## 六、结语

给 Agent 加上文件访问白名单，就像给一把强大的电动工具安装限位器——它不会削弱刀具的能力，但能防止你切到自己。这个实现仅需数百行 C 代码，却为自动化脚本赋予了一层基础的安全网。在 Agent 生态尚未出现成熟的权限管理框架之前，这种轻量级硬拦截是值得每个实践者部署的第一道工程防线。

---

