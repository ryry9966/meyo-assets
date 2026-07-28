---
title: Agent 脚本安全实战：用本地目录白名单构筑文件访问护栏
feedId: 30829
source: 综合讨论
publishedAt: 2026-07-28
---

## 背景

在 Agent 自动化实践里，脚本经常需要读写本地文件：导出分析报告、暂存截图、处理 CSV 数据、读取配置文件……这些动作一旦赋予 Agent 完整的文件系统访问权，风险就会成倍增加。一个拼写错误的路径，或者被外部输入利用的路径拼接，都可能导致关键文件被覆盖、删除或泄露。对于长时间运行、与外部工具交互的智能体，这种风险尤其需要被工程化地控制。

与其事后修补，不如一开始就限制 Agent 的操作范围：只允许它访问一个或几个受信任的目录，比如 `./workspace`、`/tmp/agent_sandbox`。这就是**目录白名单护栏**。它不替代沙箱，但足够轻量，可以直接嵌入自己编写的工具脚本或 MCP 插件中。

## 问题分析

以 Python 为例，你可能会写出这样的下载脚本：

```python
def save_report(filename, content):
    with open(filename, 'w') as f:
        f.write(content)
```

如果 `filename` 来自用户输入或大模型输出，一旦包含 `../../../etc/passwd`，就会造成路径穿越。即使你限定了 `workspace`，符号链接也可能将真实路径指向外部。简单的字符串前缀匹配根本靠不住，我们需要一个能抵抗这些花招的 **PathGuard**。

## 实现步骤

### 1. 定义白名单目录集合
白名单目录应该是**运行前明确指定的绝对路径**，可以通过环境变量或配置文件传入，例如：
```python
ALLOWED_DIRS = [
    os.path.realpath("./agent_workspace"),
    os.path.realpath("/tmp/agent_output"),
]
```

### 2. 路径规范化
所有传入路径必须经过 `os.path.realpath` 处理，它会：
- 将相对路径转为绝对路径（基于当前工作目录）
- 解析所有的 `.`、`..`
- 跟随符号链接，返回真实路径

这样就杜绝了“相对路径逃逸”和“链接跳板”的问题。

### 3. 安全判断逻辑
比较规范化后的路径是否以某个白名单目录开头，但需要注意边界：如果白名单是 `/home/user/app`，那么 `/home/user/app_evil` 不应该通过。最稳健的方法是使用 `os.path.commonpath`，并确保公共前缀等于白名单目录自身，或者用分隔符补全后再做前缀匹配。参考实现：

```python
def is_path_allowed(path: str, allowed_dirs: list) -> bool:
    real_path = os.path.realpath(path)
    for allowed in allowed_dirs:
        # 确保 real_path 就是 allowed，或者是 allowed 目录下的内容
        if os.path.commonpath([real_path, allowed]) == allowed:
            return True
    return False
```

注意：`commonpath` 不会在路径末尾添加分隔符，所以 `/home/user/app` 和 `/home/user/app_sub` 的公共路径是 `/home/user/app`，这恰恰避免了子串误判。

### 4. 包装关键文件操作
基于上面的检查函数，封装 `safe_open`、`safe_remove` 等，任何未通过检查的操作立即抛出 `PermissionError` 并记录日志。例如：

```python
def safe_open(path, mode, allowed_dirs):
    if not is_path_allowed(path, allowed_dirs):
        raise PermissionError(f"Access denied: {path}")
    return open(path, mode)
```

对于重命名、创建目录等操作也需要同样的包裹。将原始的内置函数**替换为安全版本后，Agent 脚本其余部分无需大改**，只需统一 import 这个封装模块即可。

### 5. 集成到 Agent / MCP 工具中
如果你在为 OpenClaw 编写 MCP 工具插件，可以在工具函数入口处统一拦截：
```python
@app.tool()
def write_file(filename: str, content: str):
    # 将 allowed_dirs 从环境/配置注入
    with safe_open(filename, 'w', ALLOWED_DIRS) as f:
        f.write(content)
    return "OK"
```
如果工具数量多，可以单独抽离出一个 `FileSystemAccess` 层，让所有文件工具继承或调用，避免散落的重复检查逻辑。

## 踩坑与细节

- **符号链接绕行**：务必使用 `realpath`，不要用 `abspath`。`abspath` 只拼接当前目录而不跟随链接。
- **相对路径可变性**：如果代码中有多次 `os.chdir`，相对路径的解析会不同。因此所有检查必须在**同一解析阶段**完成；最佳实践是严禁在 Agent 运行时更改工作目录。
- **Windows 兼容**：盘符和反斜杠需要特别注意。`commonpath` 在 Windows 下处理良好，但白名单目录必须使用 `realpath` 统一格式，比如 `C:\\Users\\...`。
- **性能考量**：`realpath` 会触发系统调用，如果极端频繁，可以给已检查路径加上 LRU 缓存。
- **日志与审计**：记录所有被拒绝的访问请求，作为安全事件供追溯。这有助于发现 Agent 是否存在不合理的行为模式。
- **写入与读取分离**：有时允许读取的目录比允许写入的范围更宽，可以实现两套白名单：`READ_DIRS` 和 `WRITE_DIRS`，进一步细化权限。

## 可复用建议

将上述 PathGuard 封装成一个独立的小组件，既可以直接在脚本中引用，也可以通过依赖注入的方式与 Agent 框架集成。提供一个上下文管理器来临时扩展目录白名单也很实用：

```python
with guard.temporary_allow([temp_dir]):
    # 临时允许操作 temp_dir
    process()
```

对于 OpenClaw 社区的用户，可以考虑把这种护栏作为**插件系统的基础安全层**，在工具注册时统一校验参数中的文件路径，减少每个插件开发者的重复工作。同时，在配置文件里用简单的 YAML 或环境变量定义白名单，方便在不同部署环境下快速调整。

## 总结

文件访问护栏并不复杂，却能够在 Agent 自动化的“便利”与“安全”之间划出一条清晰边界。用本地目录白名单的方式，代码侵入性低，理解成本小，而且能有效防止最常见的路径穿越和误操作。只要在工程实现中揪住路径规范化、边界处理和集成点这三个关键，就能为你的智能体脚本增加一道可靠的安全防线。

不要再让 `open()` 裸奔了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-28/7706ae1522855855.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-28/fbf38e02dc2c59b7.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-28/1cd5b8aaadd71cf7.png)

