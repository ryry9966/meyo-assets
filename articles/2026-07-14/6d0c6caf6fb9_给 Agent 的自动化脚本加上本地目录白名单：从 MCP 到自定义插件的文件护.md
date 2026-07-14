---
title: 给 Agent 的自动化脚本加上本地目录白名单：从 MCP 到自定义插件的文件护栏实践
feedId: 29087
source: 综合讨论
publishedAt: 2026-07-14
---

## 为什么需要文件访问护栏

Agent 正在越来越多地接触本地文件系统：通过 MCP 的 filesystem server 读写项目文件，利用插件批量处理文档，或由自动化脚本抓取日志、生成报告。一旦脚本没有边界，一次错误的指令就可能让 Agent 递归删除整个家目录，或把临时文件写到 `/etc` 里。

“能力越大，责任越大”在这里不是口号。给文件访问加上目录白名单，是最基础也是最有效的工程防护——限定 Agent 只能触及我们明确授权的目录，避免无意的越界。

## 常见场景与风险

- **MCP filesystem server**：官方 server 如果不配目录参数，默认行为可能是拒绝一切访问，但很多实践者为了快速跑通，会直接放开根目录或家目录。  
- **自研插件/脚本**：调用 `os.remove()`、`shutil.rmtree()` 前没有做路径校验，当用户输入包含 `../../` 时就可能逃逸到预期目录之外。  
- **自动化流水线**：Agent 在生成临时文件、解压包、移动 artifact 时，可能把关键配置覆盖或引入非预期路径。

这些风险的共同点：**代码没有内建边界，而是依赖外部输入“假设是安全的”**。

## 方案一：利用 MCP server 内建白名单

官方 [`@modelcontextprotocol/server-filesystem`](https://github.com/modelcontextprotocol/servers/tree/main/src/filesystem) 在设计时就考虑了目录限制，启动时通过 `--directory` 参数指定一个或多个允许的目录：

```bash
npx -y @modelcontextprotocol/server-filesystem \
  /home/user/allowed/project \
  /tmp/safe-workspace
```

该 server 内部会将所有文件操作路径规范化为绝对路径，再检查其是否以已配置的目录为前缀。这样即便 Agent 传入了相对路径或 `../` 试探，也会被拒绝。**这是最省心的用法**——你只需要在启动时列出白名单目录，不需要修改 server 源码。

> **注意**：如果同时需要读写和只读目录，可启动多个实例或用 `--read-only` 参数约束某些路径。权限分级同样是护栏的一部分。

## 方案二：在自定义脚本中实现路径白名单

如果你自己编写插件，或需要在 Python/Node 脚本里执行文件操作，可以加入一个轻量的校验层。

### 步骤

1. **配置白名单**：从环境变量或配置文件读取允许的目录列表（绝对路径）。
2. **统一路径规范化**：使用 `os.path.realpath()` 或 `pathlib.Path.resolve()` 将目标路径转换为规范化的绝对路径，消除 `..` 和符号链接。
3. **判断归属**：检查规范化后的路径是否以任一白名单目录为前缀。注意不要简单用字符串 `startswith`，否则 `/allowed/project` 会误匹配 `/allowed/project_evil`；应使用 `os.path.commonpath([candidate_dir, target]) == candidate_dir`。
4. **封装**：把校验逻辑写成装饰器或上下文管理器，确保每个文件操作入口都经过检查。

### 示例（Python）

```python
import os
from pathlib import Path
from functools import wraps

ALLOWED_DIRS = [
    Path("/home/user/project"),
    Path("/tmp/agent-workspace"),
]

def guard_path(func):
    @wraps(func)
    def wrapper(path, *args, **kwargs):
        resolved = Path(path).resolve()
        allowed = any(
            os.path.commonpath([str(d), str(resolved)]) == str(d)
            for d in ALLOWED_DIRS
        )
        if not allowed:
            raise PermissionError(f"Access denied: {resolved}")
        return func(str(resolved), *args, **kwargs)
    return wrapper

@guard_path
def safe_remove(path):
    os.remove(path)
```

### 踩坑点

- **符号链接陷阱**：白名单目录内部如果有符号链接指向外部，`resolve()` 会彻底解析掉。对于故意构造的链接逃逸，直接拒绝可能过于保守？这取决于安全策略。我个人实践是：**不允许符号链接指向白名单外**，并在环境初始化时审计软链接。
- **跨平台路径分隔符**：Windows 下 `commonpath` 对盘符的处理可能不一致，尽量使用 `pathlib` 统一抽象。
- **性能**：对每个文件操作都校验会产生额外开销，但相较其带来的安全性，这种开销完全可以接受。如果调用极其频繁，可缓存已校验路径的归属结果，但注意缓存失效策略（目录挂载、链接变化）。
- **白名单维护**：忘记更新白名单是最大的坑。建议将白名单配置与项目 repo 放在一起，由 CI 检查环境变量完整性，并在启动 Agent 时输出当前白名单列表以便审计。

## 可复用的工程实践

1. **容器双重隔离**：将 Agent 运行在容器内，只挂载必要的目录（如 `-v /safe/project:/workspace`），即便脚本白名单失效，也无法触碰到宿主机其他目录。
2. **只读优先**：如果脚本只需读取配置或日志，就只赋予只读权限。MCP server 支持 `--read-only`，自定义脚本也可以采用 `mode='r'` 强制限制。
3. **审计日志**：记录每次被拒绝的越界请求，便于事后排查是 Agent 幻觉还是脚本缺陷。
4. **测试边界**：编写故意的越界用例（如 `../../../etc/passwd`）来验证护栏生效，把安全测试纳入回归集。

## 总结

文件访问护栏不是新奇技术，但很容易在快速搭建 Agent 时被忽略。MCP 官方的 filesystem server 已经提供了还算完整的白名单机制，直接使用即可；而对于自定义插件，几十行路径校验逻辑就能构建起基础防线。 

核心思路只有一条：**永远不要信任外部输入，把系统允许的边界显式声明在代码里，而不是依赖默认行为**。加上容器、最小权限和审计，你的 Agent 文件操作才能真正做到可控、可追溯、不闯祸。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-14/4d1925a054f0806a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-14/bc101b7031577b86.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-14/46e0737f536c03b6.png)

