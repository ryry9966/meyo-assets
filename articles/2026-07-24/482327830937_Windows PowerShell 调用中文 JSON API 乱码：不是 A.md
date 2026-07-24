---
title: Windows PowerShell 调用中文 JSON API 乱码：不是 API 错了，是你的控制台没切 UTF-8
feedId: 30319
source: 综合讨论
publishedAt: 2026-07-24
---

## 背景：自动化脚本踩的第一个坑

在 OpenClaw 生态里，越来越多的 Agent、MCP 插件、自动化任务直接用 PowerShell 调用 REST API 并把结果喂给下游工具。中文场景下，最常遇到的现象是：`curl` 返回的 JSON 工整可读，同一个 API 用 `Invoke-RestMethod` 一打出来，名字变成了 `????` 或者 `é"??`。更麻烦的是，当脚本把返回的中文写进文件、通过管道递给 Python 或大模型时，数据已经永久损坏。

这不是 API 开发者的锅，也不是网络传输的锅——数据在内存里完好无损的瞬间，恰好是 **PowerShell 向控制台“吐字”的那一刻**，被系统默认编码截断了。

---

## 问题：为什么 PowerShell 会把中文“打坏”？

在 Windows 平台上，典型的链路是这样的：

```
网络字节流(UTF-8) → .NET 字符串(UTF-16) → 控制台输出(代码页 936/GBK) → 乱码
```

`Invoke-RestMethod` / `Invoke-WebRequest` 返回的对象中，汉字以 .NET 内部 UTF-16 形式正确存储。但当 PowerShell 要将这个字符串 **显示到控制台**，或者 **通过重定向输出到文件** 时，它就会使用 `[Console]::OutputEncoding` 所对应的编码。在中文 Windows 的 PowerShell 5.1 里，这个值默认为 Windows 代码页 936（GBK），而不是 UTF-8。于是，UTF-16 中某些 GBK 无法表示的字符（比如生僻字、全角符号、某些 Unicode 组合）就会被替换为问号或乱码。

另一个隐蔽的问题是 **重定向输出编码**。在 Windows PowerShell 5.1 中：

- `>` 和 `Out-File` 的默认编码是 `Unicode`（UTF-16 LE），并非 UTF-8；
- `Set-Content` 默认编码为 `ANSI`（即当前系统代码页，如 GBK）。

如果你不做特殊处理，即使控制台看着正常，重定向到文件后再被其它工具（要求 UTF-8）读取，依旧会变成乱码。很多 Agent 就是通过管道或临时文件交换数据，一乱码整个任务链就断了。

---

## 步骤：从诊断到根治

### 1. 确认当前环境的编码设置

在 PowerShell 5.1 中执行：

```powershell
$PSVersionTable.PSVersion
[Console]::OutputEncoding
[Console]::InputEncoding
```

典型的“问题机器”输出是：

```
PSVersion 5.1.xxxxx
OutputEncoding : System.Text.GB2312Encoding
InputEncoding  : System.Text.GB2312Encoding
```

这意味着当前控制台使用 GBK 解释输入输出。

### 2. 捕获一条中文 API，观察中间状态

用 `Invoke-RestMethod` 拉取一条带有中文字段的 JSON，例如 `{ "user": "张三", "status": "可用" }`。把结果存入变量，分两步检查：

```powershell
$resp = Invoke-RestMethod -Uri 'https://api.example.com/user/123'
# 内部存储检查（必定正常）
$resp.user
# 直接管道写文件（默认编码可能错误）
$resp | Out-File test1.txt
# 强制 UTF-8 写文件
$resp | Out-File -Encoding utf8 test2.txt
```

比较 `test1.txt` 和 `test2.txt`，`test1.txt` 用记事本打开可能看到乱码，因为文件是 UTF-16 LE，记事本需要识别 BOM；某些工具直接按字节流读取就会异常。

### 3. 临时修复：把控制台切换到 UTF-8

在脚本开头增加两行：

```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
[Console]::InputEncoding  = [System.Text.Encoding]::UTF8
```

再次执行 `$resp.user`，乱码应当消失。此时任何通过控制台输出的中文都会被正确编码为 UTF-8 传递出去。

如果必须使用 Windows PowerShell 5.1，建议在所有调用 API 的脚本模板中固化这两行设置。

### 4. 永久方案：迁移到 PowerShell 7

在 Windows 上安装 PowerShell Core (7+)，它会在启动时自动将 `OutputEncoding` 设置为 UTF-8，并且默认使用 BOM-less UTF-8 编码处理管道和文件。对于现代化的 Agent、MCP 插件工作流，**直接以 PowerShell 7 作为宿主机环境，可以从根上规避 95% 的编码陷阱**。

### 5. 发送 JSON 请求体时的字符集保护

不仅接收会乱码，**发送中文载荷时也可能被错误编码**。当你使用 `ConvertTo-Json` 生成字符串后，务必显式指定 Content-Type 的 charset：

```powershell
$body = @{ name = '张三' } | ConvertTo-Json -Compress
$params = @{
    Uri         = 'https://api.example.com/user'
    Method      = 'Post'
    Body        = $body
    ContentType = 'application/json; charset=utf-8'
}
Invoke-RestMethod @params
```

若不写 `charset=utf-8`，某些版本的 PowerShell 可能回退到 `iso-8859-1`，导致服务端收到的中文乱码。这一步在 OpenClaw 场景里尤其重要——插件向 MCP 服务器或 LLM 网关发送历史对话时，编码错误会让任务直接失败。

---

## 踩坑点：你以为修好了，其实没有

1. **字体问题伪装成编码问题**：设置 `OutputEncoding` 后中文仍然显示为方块，检查控制台字体是否支持中文字形（如“新宋体”“Consolas+中文字体回退”），这是渲染层问题，不影响真实数据。
2. **PowerShell ISE 的行为不一致**：ISE 的输出不是标准控制台，它自己的编码策略可能导致 `[Console]::OutputEncoding` 设置不生效。自动化脚本务必在 `powershell.exe` 或 `pwsh.exe` 环境中验证。
3. **管道传给外部程序**：当通过 `| python script.py` 传递中文时，PowerShell 的 `$OutputEncoding` 控制着管道传递内容的编码。如果外部程序期望 UTF-8，你同样需要设置 `$OutputEncoding` 为 UTF-8，否则管道中的数据会被编码成 GBK，Python 端大概率解析出错。
4. **BOM 陷阱**：`Out-File -Encoding utf8` 在 Windows PowerShell 5.1 下会在文件开头写入 BOM (0xEFBBBF)。某些 Linux 工具或 JSON 解析器对 BOM 敏感，可能导致解析失败。根治办法是直接使用 PowerShell 7 的 `-Encoding utf8NoBOM`，或在 5.1 下用 `[System.IO.File]::WriteAllText("file.json", $content, [System.Text.UTF8Encoding]::new($false))` 写无 BOM 文件。

---

## 可复用建议：构建编码免疫的自动化脚本

在 OpenClaw/Agent 类项目里，建议直接锁定运行环境为 **PowerShell 7**，并在脚本头部统一引入一段初始化代码块：

```powershell
# 初始化 UTF-8 环境（兼容 5.1 和 7）
if ($PSVersionTable.PSVersion.Major -lt 6) {
    [Console]::OutputEncoding = [System.Text.Encoding]::UTF8
    [Console]::InputEncoding  = [System.Text.Encoding]::UTF8
    $OutputEncoding = [System.Text.Encoding]::UTF8
}
$PSDefaultParameterValues['Out-File:Encoding'] = 'utf8'
$PSDefaultParameterValues['*:Encoding'] = 'utf8'
```

这样，即便遗留环境没有升级 PowerShell，脚本也能自动适配，把所有输出、管道、文件编码收敛到 UTF-8。插件开发者可以把这段逻辑打包成模块的启动脚本，避免用户侧的低级编码故障。

---

## 总结

Windows PowerShell 把中文 JSON 打坏，不是 API 坏了，不是网络传坏了，而是 **统一字符集 (Unicode) 与 Windows 遗留代码页之间的兼容层在某一步失效了**。根本原因在于 PowerShell 5.1 控制台输出默认依赖系统代码页（GBK），而现代 API 和工具链全部基于 UTF-8。

根治路径简单直接：
- 诊断时检查 `[Console]::OutputEncoding`；
- 治疗时设定为 UTF-8，并统一文件与管道编码；
- 长期方案切换到 PowerShell 7，让默认值天然正确。

在这个自动化脚本跑在每一个 Agent 缝隙里的时代，编码问题一劳永逸地解决，才能让中文场景的 MCP 调用、知识库写入、LLM 对话记录不再“断字”。工程上的安全感，往往就藏在这两行看似不起眼的 UTF-8 声明里。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-24/aef6728d43686a82.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-24/46ef75432db9b930.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-24/75a3aa395e3f3435.png)

