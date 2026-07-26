---
title: PowerShell 处理中文 JSON 的编码陷阱与工程化修复
feedId: 30631
source: 综合讨论
publishedAt: 2026-07-27
---

## 背景

在 Windows 环境下，大量自动化脚本通过 PowerShell 调用以 JSON 格式返回数据的 HTTP API。OpenClaw 工具链、MCP 插件、CI/CD 胶水脚本中经常出现以下模式：

```powershell
$resp = Invoke-RestMethod -Uri "https://api.example.com/data" -Method Get
$resp.data.name   # 中文？ 问号、方块、锟斤拷
```

此时控制台输出、写文件或通过管道传给下游 Python/Node 程序后，中文信息变成乱码。问题并不在于 API 响应本身，而在于 PowerShell 的编码处理链路。本文梳理这一链路，并提供在 2025 年仍然有效的工程化修复方案。

## 问题定位

Windows 上的 PowerShell 有两个主要版本：旧版 Windows PowerShell 5.1（内置），和新版 PowerShell 7+（可独立安装）。二者行为差异显著，但许多生产环境仍锁定在 5.1。下面以最易出问题的 Windows PowerShell 5.1 为对象分析。

### Invoke-RestMethod 已经正确解码

`Invoke-RestMethod` 会识别响应头的 `charset`（通常为 UTF-8），将字节流解码为 .NET 字符串返回。因此 `$resp` 内部的字符串是正确保存了中文字符的。

### 显示与输出的编码陷阱

- **控制台显示**：在 Windows 终端（conhost 或 Windows Terminal）中，PowerShell 宿主本身支持 Unicode 显示，直接输入 `$resp.data.name` 并回车一般能正常显示，但一旦将输出重定向到文件或通过管道传给外部命令，问题立刻暴露。
- **`>` 重定向**：Windows PowerShell 5.1 中，`>` 操作符使用 **UTF-16 LE** 编码写入文件。这会导致许多仅按 UTF-8 解析的工具显示乱码。
- **`Out-File` 和 `Set-Content` 默认编码**：5.1 中 `Out-File` 和 `Set-Content` 默认使用 **Unicode (UTF-16LE)** 或 **ASCII**（取决于参数），而非 UTF-8。保存 JSON 时尤为致命。
- **`ConvertTo-Json` 的非 ASCII 转义**：Windows PowerShell 5.1 的 `ConvertTo-Json` 会强制将所有非 ASCII 字符转义为 `\uXXXX` 格式。即便用 UTF-8 保存文件，文件中也是转义序列而非原始中文。虽然 JSON 标准允许这种写法，但人工阅读和部分解析器调试极其不便。
- **管道和外部命令的编码**：当把数据通过管道传递给 Python 脚本或作为 OpenClaw 工具的参数时，PowerShell 使用 `$OutputEncoding` 将字符串编码为字节流传给子进程。在 Windows PowerShell 5.1 中，`$OutputEncoding` 默认为 **ASCII**，导致中文变成 `?`。

## 工程化修复步骤

以下步骤基于 Windows PowerShell 5.1，若使用 PowerShell 7+，多数编码默认早已是 UTF-8，但仍可作为防御性配置。

### 1. 设置全局输出编码

在脚本开头强制设置控制台输出编码和管道编码为 UTF-8：

```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$OutputEncoding = [System.Text.Encoding]::UTF8
```

- `[Console]::OutputEncoding` 影响控制台显示和部分写入操作。
- `$OutputEncoding` 影响将字符串传递给外部命令时的编码。

### 2. 安全获取 JSON 并写文件

避免直接 `> file.json`，使用 `Out-File -Encoding utf8` 或 `Set-Content -Encoding utf8`：

```powershell
$resp = Invoke-RestMethod -Uri "https://api.example.com/data" -ContentType "application/json; charset=utf-8"
# 将对象重新序列化为 JSON 并保存为 UTF-8 可读文件（接受转义字符）
$resp | ConvertTo-Json -Depth 10 | Out-File -FilePath "data.json" -Encoding utf8
```

如果下游程序必须读取原始中文（不接受 `\uXXXX` 转义），且你无法升级到 PowerShell 7，则需要使用 .NET 类绕过 `ConvertTo-Json` 的转义行为：

```powershell
$jsonString = $resp | ConvertTo-Json -Depth 10 -Compress
$jsonString = [System.Text.RegularExpressions.Regex]::Unescape($jsonString)
$jsonString | Out-File -FilePath "data_unescaped.json" -Encoding utf8
```

`Regex.Unescape` 会把 `\uXXXX` 变回实际字符，但副作用是所有符合 `\u....` 的模式都会被反转义，一般 JSON 字符串中不会冲突，可放心使用。

### 3. 安全传递参数给外部程序

当通过 `Start-Process` 或直接调用可执行文件传递中文参数时，除设置 `$OutputEncoding` 外，还可以将字符串写入临时文件，让子进程从文件读取，彻底避免编码协商问题。OpenClaw 插件设计时可优先考虑文件传递复杂数据，而不是命令行参数。

### 4. 在 OpenClaw 或 MCP 脚本中的最佳实践

如果你的 PowerShell 脚本被 OpenClaw 通过 child_process 调用，父进程通常会设置 `-Command` 并捕获标准输出。此时在你的脚本中应显式输出 UTF-8 并确保退出前清理编码状态：

```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
Write-Output $result
```

同时，在 OpenClaw 配置中调用脚本时，可优先使用 `pwsh.exe`（PowerShell Core）而不是 `powershell.exe`，新版默认输出编码即为 UTF-8，可减少大量兼容性代码。

## 典型踩坑案例

**案例 1：日志文件中文变问号**

一位开发者在 Windows Server 2016 上编写定时任务脚本，用 `Invoke-WebRequest` 拉取天气 API，`Content` 写入日志文件：

```powershell
$content = Invoke-WebRequest $url | Select-Object -ExpandProperty Content
$content > log.txt
```

因 `>` 输出 UTF-16，本地编辑器默认按 UTF-8 打开显示乱码。修改为 `$content | Out-File -Encoding utf8` 解决。

**案例 2：ConvertTo-Json 后 Webhook 接收错误**

机器人插件将内存对象转为 JSON 通过 webhook 发出，接收端因 `\uXXXX` 转义导致字符串长度校验失败。最初尝试手写 JSON 拼接，后期迁移至 PowerShell 7 并设置 `-EscapeHandling EscapeNonAscii` 避免转义。

## 可复用编码安全代码块

将以下代码块放在所有 Windows PowerShell 脚本顶部，可防御 90% 的中文乱码：

```powershell
# ---- 编码安全模板 ----
if ($PSVersionTable.PSVersion.Major -lt 6) {
    [Console]::OutputEncoding = [System.Text.Encoding]::UTF8
    $OutputEncoding = [System.Text.Encoding]::UTF8
    $PSDefaultParameterValues['*:Encoding'] = 'utf8'
}
```

使用 `$PSDefaultParameterValues` 可以为所有支持 `-Encoding` 参数的 cmdlet（如 `Out-File`、`Set-Content`、`Export-Csv` 等）设置默认 UTF-8 编码，避免逐个指定。

## 总结

Windows 环境下的 PowerShell 中文 JSON 乱码，本质是两套编码体系的不匹配：网络响应通常为 UTF-8，而 Windows PowerShell 的历史包袱（OEM 代码页、UTF-16 重定向、ASCII 管道）使数据流在多个环节发生转换。通过显式设置控制台编码、输出编码、文件操作编码，并理解 `ConvertTo-Json` 的转义行为，可以稳定地在自动化工作流中传递中文。OpenClaw 开发者在编写插件时，应将编码安全意识作为工程化基础，优先使用 PowerShell 7，并在必须兼容 5.1 时采用上述模板，避免因字符乱码导致的数据处理链中断。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-27/5c165f74ac8a0ce8.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-27/e8b8363a2bcb4e35.png)

