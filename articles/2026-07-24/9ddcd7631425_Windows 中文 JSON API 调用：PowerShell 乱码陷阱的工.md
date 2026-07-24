---
title: Windows 中文 JSON API 调用：PowerShell 乱码陷阱的工程化解决方案
feedId: 30268
source: 综合讨论
publishedAt: 2026-07-24
---

## 背景

在 Windows 上用 PowerShell 调用返回中文内容的 JSON API 时，开发者常常发现，明明服务端返回的 `Content-Type: application/json; charset=utf-8` 是规范的，用 `curl.exe` 或 Git Bash 请求能看到正确的中文，但一进入 PowerShell 环境——无论是直接 `Invoke-RestMethod` 还是通过管道输出到文件——中文就变成了乱码，常见如 `鎴戠殑`、`æˆ‘çš„` 这类西欧编码产物。这不仅阻碍了脚本化数据处理，更让 OpenClaw、Agent、MCP 等自动化流程在 Windows 节点上大面积失效。

## 问题根源

PowerShell 是一个面向对象的 Shell，它对文本流的处理依赖于当前会话的编码设置。Windows PowerShell 5.1（基于 .NET Framework）和 PowerShell Core 7+ 的行为有细微差异，但核心问题一致：**控制台输出编码 (`[Console]::OutputEncoding`) 与脚本或管道的外部编码不匹配，且 `Invoke-WebRequest`/`Invoke-RestMethod` 在解析 HTTP 响应时，如果未明确指定 UTF-8，会倾向使用系统区域设置的默认编码（如 GBK、CP936 或 ISO-8859-1）。**

具体表现：

- 直接运行 `Invoke-RestMethod https://example.com/api`，在控制台窗口看到乱码，但对象属性是正确的（因为 PowerShell 内部用 Unicode 存储字符串）。
- 将结果通过 `| Out-File output.json` 写入磁盘，文件内容乱码，因为 `Out-File` 默认使用 Unicode (UTF-16LE) 或 ASCII（取决于 PowerShell 版本），且管道传递时可能被重新编码。
- 将输出通过 `> output.json` 重定向，几乎必乱码，因为重定向操作符直接使用 `[Console]::OutputEncoding` 指定的编码（通常是 OEM 编码，如 CP437 或 CP850），而其中没有中文码点。

## 验证与修复步骤

### 1. 检查当前编码状态
```powershell
[Console]::OutputEncoding.EncodingName   # 输出类似 Western European (ISO) 或 OEM United States
$OutputEncoding                          # 同上，反映管道编码
[System.Text.Encoding]::Default.EncodingName  # 通常为 GB2312（中文系统）或 Windows-1252
```
典型故障环境：`[Console]::OutputEncoding` 为 `US-ASCII` 或 `Western European (ISO)`。

### 2. 临时修复：设置为 UTF-8
在执行 API 调用前，强制修改会话编码：
```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$OutputEncoding = [System.Text.Encoding]::UTF8
```
然后重新运行请求：
```powershell
$resp = Invoke-RestMethod -Uri "https://api.example.com/data" -ContentType "application/json; charset=utf-8"
$resp | ConvertTo-Json -Depth 3 | Out-File -FilePath "data.json" -Encoding utf8
```
注意：`Out-File` 必须显式带上 `-Encoding utf8`，`Set-Content` 同理。`ConvertTo-Json` 默认不会破坏中文，但某些旧版本会将其转义为 `\uXXXX`，可通过 `-EscapeHandling EscapeNonAscii` 控制（PSCore 6.2+ 支持）。

### 3. 永久配置（存入 Profile）
在 `$PROFILE` 文件中添加初始化编码：
```powershell
# For Windows PowerShell 5.1
$pshome\profile.ps1  (AllUsersAllHosts)
# 或当前用户
if (-not (Test-Path $PROFILE)) { New-Item -Type File -Path $PROFILE -Force }
Add-Content -Path $PROFILE -Value @'
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$OutputEncoding = [System.Text.Encoding]::UTF8
'@
```
对于 PowerShell Core (7+)，UTF-8 通常是默认的，但仍建议检查。

### 4. 使用 curl.exe 的坑（混用场景）
很多用户在 PowerShell 里调用 Git 自带的 `curl.exe`（别名可能冲突），它输出 UTF-8 到 stdout，但 PowerShell 用它时捕获输出会按 `[Console]::OutputEncoding` 解码，需这样调用：
```powershell
$json = & curl.exe -s https://api.example.com/data
$json = [System.Text.Encoding]::UTF8.GetString([System.Text.Encoding]::Default.GetBytes($json))
```
更直接的方法是使用 `Start-Process` 配合重定向，避免编码劫持。

## 踩坑要点

- **PSCore 与 Windows PowerShell 的差异**：Core 版默认 `$OutputEncoding` 为 UTF-8，但 Windows 控制台可能仍使用遗留编码，需单独设 `[Console]::OutputEncoding`。
- **BOM 问题**：UTF-8 with BOM 可能导致 API 服务端解析失败。用 `-Encoding utf8NoBOM` 写入（PSCore 6+），或通过 `[System.IO.File]::WriteAllText("path", $string, [System.Text.UTF8Encoding]::new($false))` 实现无 BOM UTF-8。
- **管道传递的隐式转换**：`$result | some-cmdlet` 中，若 `some-cmdlet` 将对象转为字符串，可能触发区域编码，应先将变量转为 JSON 字符串，再传递。
- **Invoke-RestMethod 乱码返回值**：少数 API 无视 `Accept-Charset` 头，直接返回非 UTF-8 内容。可先用 `Invoke-WebRequest` 获取字节流，再手动解码：`[System.Text.Encoding]::UTF8.GetString($response.Content)`。

## 可复用建议

1. **封装请求函数**：将编码设置与请求封装成模块函数，确保每次调用前强制设置 UTF-8 并处理解码，避免散落的编码调整。
2. **标准化文件名与内容处理**：凡是落地 JSON 文件，统一使用 `Set-Content -Encoding utf8NoBOM` 或 `Out-File -Encoding utf8`，杜绝系统默认。
3. **在 Agent/MCP 节点中预置编码脚本**：Windows 节点启动 agent 时，自动执行 profile 设置，确保自动化上下文不乱码。
4. **验证模式**：对下载的 JSON 文件增加快速编码验证，利用 `[System.Text.Encoding]::UTF8.GetString([System.IO.File]::ReadAllBytes($file))` 检查是否可逆解析，提前发现乱码。

## 总结

PowerShell 中文乱码本质上是控制台编码、管道编码与文件输出编码三重不一致造成的。解决方案并不复杂：**将 `[Console]::OutputEncoding` 和 `$OutputEncoding` 固定为 UTF-8，并配合显式的文件输出编码参数**。这种工程化处置能让 Windows 上的 JSON API 调用在 OpenClaw 插件、MCP 工具链和自动化脚本中稳定运行。记住，在脚本世界，中文不是原罪，默认编码才是。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-24/18292d0b73637da5.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-24/dc583748a969413c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-24/4e1e037152610b49.png)

