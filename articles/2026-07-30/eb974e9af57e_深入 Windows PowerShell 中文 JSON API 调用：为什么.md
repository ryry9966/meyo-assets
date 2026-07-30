---
title: 深入 Windows PowerShell 中文 JSON API 调用：为什么输出会“打坏”汉字
feedId: 31011
source: 综合讨论
publishedAt: 2026-07-30
---

## 背景：当 Agent 寄出“锟斤拷”

如果你正用 OpenClaw 或其他 Agent/MCP 工具在 Windows 上搭建自动化链路，很可能遇到这样一种情况：插件通过 PowerShell 调用某个 JSON API，返回的中文日志变成了 `????`，或者 API 收到了乱码内容，而同样的请求在 Linux、macOS 或 Postman 里一切正常。

这不是幻觉，而是 Windows 下 PowerShell 的编码处理与 JSON/HTTP 标准之间的一个真实“摩擦力”。本文聚焦于 **Windows PowerShell 5.1（桌面版）** 在这一问题上的表现，给出可复现、可修复的工程化方案，同时也讨论 PowerShell Core (7+) 的改进点，帮助你在 Agent 工作流中避免非显式的数据损坏。

## 问题表现：从管道到 HTTP 体，中文被悄悄改写

典型的现象包括：

- 用 `ConvertTo-Json` 将一个包含中文的 PowerShell 对象转化为 JSON 字符串时，中文变成了 `\uXXXX` 这种 Unicode 转义序列。这在 JSON 标准中虽合法，但部分服务端（特别是古老或严格解析的实现）可能无法正确还原，或者日志里不再可读。
- 通过 `Invoke-RestMethod` 或 `Invoke-WebRequest` POST JSON 数据时，若没有显式指定 UTF-8，`-Body` 参数中的中文会被以系统活动代码页（如 GBK）编码，导致服务端收到乱码。
- 使用 `Out-File` 或 `>` 重定向保存 JSON 时，中文变成乱码，而文件本身用 Notepad++ 看编码是 “UTF-16 LE” 或 “GB2312”。
- 部署在 Windows 上的 MCP 服务或 Agent 插件，从 CLI 工具（如 `curl.exe`）切换至 PowerShell 封装后，输入参数的汉字被“吃”掉。

根本原因在于 **Windows PowerShell 的默认输出编码不是 UTF-8**，而 JSON/HTTP API 的最佳实践几乎都要求 UTF-8。

## 根因分析

**1. `ConvertTo-Json` 的转义行为**
在 Windows PowerShell 5.1 中，`ConvertTo-Json` 默认会将非 ASCII 字符转义为 Unicode 转义序列。虽然这是合法的 JSON，但当你需要人类可读的日志，或者下游服务不接受 `\u` 转义时（极少见但仍存在），就会出问题。更关键的是，如果你将结果通过管道传给其他命令，编码会在管道中被再次转换。

**2. 管道和文件输出的编码**
`>` 和 `Out-File` 默认使用 UTF-16 LE 编码（Unicode），这在类 Unix 工具和绝大多数 HTTP 客户端看来是“典型问题”。`Set-Content` 默认使用系统活动代码页（中文 Windows 是 GBK）。因此你保存一个 JSON 文件，哪怕内容在屏幕上显示正常，实际写入的字节早已不是 UTF-8。

**3. `Invoke-RestMethod` 的 Content-Type 缺位**
当你发送 `-Body` 时，PowerShell 不会自动为你添加 `charset=utf-8`。即使 `$body` 是字符串，如果该字符串内部编码与网络流的编码不一致，就可能被错误序列化。典型场景：先用 `ConvertTo-Json` 生成 JSON 串（此时字符串在内存中是 .NET 字符串，编码无关），但随后传值给 `Invoke-RestMethod` 时，如果没有显式设置 `ContentType`，PowerShell 可能会使用 `application/x-www-form-urlencoded` 或者省略 charset，导致服务端按照默认 ISO-8859-1 或其它编码解析，中文报废。

**4. Windows 控制台的代码页**
在 PowerShell 控制台内运行的脚本，其输出通过 `[Console]::OutputEncoding` 决定。这个值常常与系统 OEM 代码页一致，而非 UTF-8。当你的 JSON 结果被打印到控制台并被 Agent 捕获时（例如通过 stdout 读取），Agent 如果假设是 UTF-8 流，中文同样会乱码。

## 工程化修复步骤

下面的操作按推荐顺序给出，适用于 **Windows PowerShell 5.1** 环境。如果你的生产环境可以升级，优先考虑 PowerShell Core，但这不总是可行。

### 1. 确保所有文本 IO 使用 UTF-8

在脚本开头强制设定输出编码：

```powershell
$OutputEncoding = [System.Text.Encoding]::UTF8
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$PSDefaultParameterValues['*:Encoding'] = 'utf8'
```

这一步可以解决大部分 `Out-File`、`Set-Content` 以及管道输出的编码问题。但对于 `>` 重定向无效（`>` 由 cmd 接管），所以推荐改用 `Set-Content -Path ... -Encoding UTF8` 或 `Out-File -Encoding UTF8`。

### 2. 构造 JSON 时避免非必要转义，确保可读性

如果你需要 JSON 体保持原文可读（例如用于调试），可以使用 `ConvertTo-Json` 然后反序列化再序列化？不，那样成本高。更简单的方式：直接使用 .NET 的 `System.Text.Json` 或 `Newtonsoft.Json`，显式控制序列化选项。如果坚持使用内置 cmdlet，PowerShell 5.1 没有直接参数关闭 Unicode 转义，不过你可以考虑：

```powershell
$obj | ConvertTo-Json -Depth 10 | ForEach-Object { [System.Text.RegularExpressions.Regex]::Unescape($_) }
```

但这只是后处理，会将 `\uXXXX` 转回字符，前提是字符串仍然是 .NET 字符串，无编码损失。

**工程师建议**：对于 Agent 自动化，不必强求人类可读的 JSON 中间文件，直接保证 “端到端 UTF-8” 即可。

### 3. 发送 HTTP 请求时显式指定 UTF-8

不要依赖默认值。写法如下：

```powershell
$body = @{ message = "你好世界" } | ConvertTo-Json -Compress
$headers = @{
    "Content-Type" = "application/json; charset=utf-8"
}
Invoke-RestMethod -Uri "http://api.example.com" -Method Post -Body $body -Headers $headers -ContentType "application/json; charset=utf-8"
```

`-ContentType` 参数会覆盖 header 中的同名项，直接传 `application/json; charset=utf-8` 最保险。

### 4. 若在 MCP 或 Agent 的主进程中捕获 PowerShell 输出

如果 Agent 通过 `powershell.exe -Command "..."` 启动并捕捉 stdout，需要让 PowerShell 以 UTF-8 模式输出。全局环境变量 `$env:PYTHONUTF8=1` 等是 Python 的，对 PS 不适用。可以用启动参数：

```
powershell.exe -NoProfile -Command "[Console]::OutputEncoding=[Text.Encoding]::UTF8; .\script.ps1"
```

或者在脚本内强制执行编码。还需确认 Agent 读取管道时使用 UTF-8 解码，例如在 Python 的 `subprocess` 中指定 `encoding='utf-8'`。这样双方才对齐。

## 踩坑点：你以为改了，其实没改

- **`$OutputEncoding` 不影响 `Write-Host`**。`Write-Host` 绕过标准流，使用的是控制台主机编码。如果要传数据给下游，避免用 `Write-Host`，使用 `Write-Output`。
- **`chcp 65001` 不够**。在 PowerShell 内执行 `chcp 65001` 只改变控制台代码页，不改变 `[Console]::OutputEncoding`，也不影响文件写入编码。仍需同时设定 .NET 对象。
- **BOM 陷阱**：用 `Out-File -Encoding UTF8` 会输出带 BOM 的 UTF-8（Windows 风格）。部分 JSON API 服务端拒绝 BOM，解析失败。若不需要 BOM，使用 `[System.IO.File]::WriteAllText($path, $json, [System.Text.UTF8Encoding]::new($false))` 写纯 UTF-8。
- **PowerShell Core 的“隐性升级”**：在 VSCode 或某些现代终端里，如果默认 shell 是 PS7，编码问题会减轻不少。但你的 Agent 部署到目标机器可能仍是 PS5.1，必须在测试环境保持一致。

## 可复用建议清单

1. **优先换用 PowerShell 7**：它默认输出 UTF-8 no BOM，`ConvertTo-Json` 也升级了。如果无法换，必须全程显式编码。
2. **统一函数封装**：编写一个 `Invoke-MyApi` 函数，内置 Content-Type 和编码逻辑，避免每次手工指定。
3. **团队约定**：凡 API 交互，必须写 `Content-Type: application/json; charset=utf-8`。
4. **文件输出一律采用 `Set-Content -Encoding UTF8NoBOM`**（PS5.1 没有 `UTF8NoBOM` 选项，需要调用 .NET 方法，如上）。
5. **监控 Agent 日志**：在开发阶段记录发出的实际字节（如 `[System.Text.Encoding]::UTF8.GetBytes($body) | Format-Hex`），快速判定问题是在 JSON 构造、网络传输还是接收侧。

## 总结

Windows PowerShell 把中文 JSON 打坏，不是 PowerShell “不支持 Unicode”，而是层层默认的编码安利和不透明的字符集协商让你在不经意间引入了破坏。对于构建在 OpenClaw 这类 Agent 框架上的自动化工具来说，编码缝隙会在调用链上被放大，最终变成 API 返回 400 或更隐蔽的逻辑错误。

根治的办法是 **用工程纪律对抗默认值**：在脚本顶部锁定 UTF-8，在每次 HTTP 会话中显式声明，在文件 IO 中放弃便捷重定向，转向明确指定编码的 API。当发现隔壁同事的 Windows 机器上又出现乱码时，不用再花两小时抓鬼，把这篇文章丢过去即可。

---

