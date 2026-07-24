---
title: Windows 里 PowerShell 调 API，中文被打坏的排查与根治
feedId: 30318
source: 综合讨论
publishedAt: 2026-07-24
---

## 背景：当自动化脚本撞上中文乱码

在 OpenClaw、Agent 工具链、MCP 插件等自动化实践中，我们经常在 Windows 上用 PowerShell 快速调用 REST API 获取数据。比如抓取一个包含中文标题的 JSON 响应、调用智能体上游服务，或者用 `Invoke-RestMethod` 把结果传给下一个模块。脚本逻辑明明是对的，但一旦遇到中文，控制台就变成一串 `?` 或者看不出原形的“火星文”。更致命的是，如果你把这种“表面乱码”的字符串又重新编码、写了文件或喂给下游，数据就真的被永久打坏了。

这个问题藏在 Windows 控制台编码、PowerShell 的默认行为以及 JSON 处理细节里，表面看起来像 API 返回了脏数据，其实完全是本机环境编码没对齐。下面我会从一次真实的翻车说起，把根因、急救方案和可工程化的规范都讲清楚。

## 问题的典型症状

在一个 Agent 插件里，我需要用 PowerShell 调用某个内部服务，返回的 JSON 长这样：

```json
{ "status": "ok", "city": "北京", "temp": 5 }
```

最简单的调用方式：

```powershell
$resp = Invoke-RestMethod -Uri "https://api.example.com/weather"
$resp.city
```

在 Windows PowerShell 5.1 的终端里运行，直接输出显示为 `??` 或 `縺薙�` 这样的乱码。但如果用 `Write-Host $resp.city` 有时又能正确显示中文；而一旦把对象转成 JSON 字符串再用 `Write-Output` 输出，乱码又回来了。更诡异的是，在 VS Code 的 PowerShell 集成终端里可能一切正常，但在 CICD 或者 Windows Task Scheduler 里就坏。

这背后其实是两套编码在打架：.NET 内部字符串（UTF-16）是完好的，但 PowerShell 主机会把输出按 **控制台输出编码** 转换成特定字节流，而你的终端窗口又用自己的编码去解释这些字节。当这三者不全对齐 UTF-8 时，中文就碎了。

## 根因分析：控制台编码 != 字符串编码

PowerShell 的 `Invoke-RestMethod` 会自动解析 JSON，返回的 `$resp.city` 是一个 .NET 字符串，底层是 Unicode，本身没有问题。问题发生在 **将字符串写到控制台** 这一环节。

Windows 控制台（conhost）有一套独立的输入/输出编码：

- **输出编码**由 `[Console]::OutputEncoding` 决定，表示控制台在显示时，将 .NET 字符串编码成什么字节序列传给终端缓冲。
- **终端本身的代码页**（例如 `chcp` 看到的 936 GBK/65001 UTF-8）决定如何把字节序再渲染成字符。

Windows PowerShell 5.1 默认的 `[Console]::OutputEncoding` 通常跟随系统区域设置，中文系统下多为 GBK（代码页 936），但你的终端窗口可能已经被 VS Code、Windows Terminal 等配置成 UTF-8（代码页 65001）。两端不一致时，GBK 编码的字节被当成 UTF-8 解释，就会出现“烫烫烫”“锟斤拷”这类经典乱码。

PowerShell Core（7+）默认的 `[Console]::OutputEncoding` 会努力设为 UTF-8，因此这类问题大幅减少。但还有很多生产环境依然使用内置的 Windows PowerShell 5.1，而各种 Agent 工具链、插件系统也常常把它作为默认解释器。

另外，`$OutputEncoding` 是另一个坑：它决定了通过管道（`|`）传给外部命令时使用的编码，如果设为默认的 ASCII（US-ASCII），那么中文字符在传给 `python`、`curl` 等外部进程时会直接被丢弃或转为 `?`。

## 急救与标准化修复步骤

### 1. 诊断当前编码

先搞清楚当前环境的编码状态：

```powershell
# 控制台输入/输出编码
[Console]::InputEncoding.EncodingName
[Console]::OutputEncoding.EncodingName

# 当前终端代码页
chcp

# 管道输出编码
$OutputEncoding.EncodingName
```

在 Windows PowerShell 5.1 中文系统上，`OutputEncoding` 多半显示 `Chinese Simplified (GB2312)` 或 `GBK`；代码页可能是 `936`；`$OutputEncoding` 很可能是 `US-ASCII`。

### 2. 让控制台输出使用 UTF-8

在脚本开头强制设置输出编码为 UTF-8，并同步调整终端代码页：

```powershell
# 将控制台输出编码改为 UTF-8
[Console]::OutputEncoding = [Text.Encoding]::UTF8
# 调整终端代码页为 UTF-8 (65001)
chcp 65001 > $null
```

这样 `Write-Output`、`Write-Host`、直接输出字符串都会以 UTF-8 字节流发送到终端，终端再以 UTF-8 解码就能正确显示。

**但注意：** 有些旧版 conhost 对代码页 65001 支持有 bug，可能出现光标混乱、退格异常。如果只是做自动化脚本（不需要人工交互），这个副作用可以忽略。

### 3. 守护管道输出编码

如果数据要通过管道传给外部命令，必须同步修改 `$OutputEncoding`：

```powershell
$OutputEncoding = [Text.Encoding]::UTF8
```

这样像 `$jsonString | python script.py` 时，Python 端 `sys.stdin.read()` 拿到的就是正确的 UTF-8 字节流，不会丢中文。

### 4. 安全地保存到文件

不要用 `>` 或 `Out-File` 的默认行为，因为它们会沿用控制台编码。显式指定 UTF-8：

```powershell
$resp | ConvertTo-Json -Depth 10 | Out-File -Encoding utf8 result.json
```

或使用 `Set-Content -Encoding utf8`。

### 5. 确认整个工具链的编码意识

如果你在 OpenClaw 插件或 MCP 服务器中调用 PowerShell 子进程，那父进程读取 `stdout` 时也必须假定子进程输出是 UTF-8。可以在启动 PowerShell 时通过 `-Command` 执行初始化代码，或者预先写一个 profile 脚本，保证每次执行都自动应用上述设置。

例如在 Agent 里调用：

```powershell
powershell.exe -NoProfile -Command `
  "[Console]::OutputEncoding=[Text.Encoding]::UTF8; chcp 65001>$null; $OutputEncoding=[Text.Encoding]::UTF8; Invoke-RestMethod 'https://api.example.com/data'"
```

这样即使外部 Agent 是以二进制流读取 stdout，只要按 UTF-8 解码，中文就不会坏。

## 踩坑点与小细节

- **只改 `chcp` 不顶用**：你即使执行了 `chcp 65001`，如果没有修改 `[Console]::OutputEncoding`，PowerShell 依然会用 GBK 把 Unicode 字符串转成字节，只是终端尝试用 UTF-8 解释这些 GBK 字节，乱码依旧。
- **VS Code 终端为什么正常？** VS Code 的 PowerShell 扩展会通过一些内部机制（PSReadLine、设置环境变量等）强制输出 UTF-8，所以开发环境下问题被隐藏，上线到裸 PowerShell 时才暴露。
- **Invoke-RestMethod 返回的对象不要过度序列化**：如果你只需要取一两个字段，直接用 `$obj.city`，不要随手转成 JSON 字符串输出。即使用 `ConvertTo-Json`，也记得 `-Compress` 再加上 `-Depth` 避免截断，然后通过 UTF-8 管道或文件输出。
- **PowerShell Core 可以根治**：条件允许的话，直接把解释器从 `powershell.exe` 换成 `pwsh.exe`，它默认就是 UTF-8 世界，你只要处理好 `$OutputEncoding` 就可以了。

## 可复用的工程建议

对于 OpenClaw 社区的自动化实践者，建议把以下代码块放进你所有 Windows PowerShell 脚本的“前八行”：

```powershell
if ($PSVersionTable.PSVersion.Major -lt 6) {
    [Console]::OutputEncoding = [Text.Encoding]::UTF8
    chcp 65001 > $null
}
$OutputEncoding = [Text.Encoding]::UTF8
```

同时在插件文档里注明：如果你的插件在 Windows 上运行且通过 PowerShell 子进程调用外部 API，请确保已设置以上编码。更彻底的做法是用 `pwsh` 作为默认 shell，一劳永逸。

## 总结

Windows 下 PowerShell 调用中文 JSON API 出现乱码，本质是控制台编码、终端代码页和 .NET 字符串之间的多层转换没有统一到 UTF-8。这锅不该让 API 或 JSON 解析库背，而应该用三行标准配置解决。对于 Agent、MCP 这类高度依赖进程间通信与自动化流程的场景，编码一致性是可靠性的基石。把它写进模板，比每次追着乱码“抢救”要划算得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-24/51839582c83e5154.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-24/bd9431dfe541ce66.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-24/e3821f23f495d519.png)

