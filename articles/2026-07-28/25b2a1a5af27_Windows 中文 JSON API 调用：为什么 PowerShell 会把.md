---
title: Windows 中文 JSON API 调用：为什么 PowerShell 会把中文打坏｜排障与可复用脚本
feedId: 30760
source: 综合讨论
publishedAt: 2026-07-28
---

# Windows 中文 JSON API 调用：为什么 PowerShell 会把中文打坏

## 背景

在 Windows 上用 PowerShell 调用外部 JSON API 是本地自动化、数据采集、MCP 工具开发的常见做法。只要 API 返回里带一点中文——用户名、地址、备注——就很容易遇到：

- `Invoke-RestMethod` 返回的中文变成 `?` 或乱码
- 准备好的 JSON body 发出去，服务端收到的中文全变成 `????`
- 重定向输出的 JSON 文件用 VS Code 打开正常，用记事本或者其它工具打开却显示乱码

这类问题往往不是 API 本身的锅，而是 PowerShell 在与 Windows 控制台、文件输出打交道时的编码行为不一致造成的。本文梳理最易踩坑的编码链路，给出诊断方法和工程化的可复用方案。

## 问题本质：PowerShell 的编码不是 UTF-8

Windows 原生控制台通常使用系统区域设置对应的代码页，例如简体中文是代码页 936 (GBK)。而现代 Web API 几乎统一使用 UTF-8。

PowerShell 5.x 及更早版本在许多场景下并不以 UTF-8 作为默认编码：

- `Out-File` 和重定向运算符 `>` 默认输出 **UTF-16 LE with BOM**，对大多数文本工具不可见的中文会变成双字节且带 BOM
- `Invoke-WebRequest` / `Invoke-RestMethod` 在构造请求体时，可能沿用 `$OutputEncoding` 变量（通常为 ASCII）来编码字符串，导致中文直接丢失
- `[Console]::OutputEncoding` 和 chcp 不同步时，控制台打印 JSON 就会看到乱码

因此，即使你肉眼看到字符串是中文，经过序列化和传输后也可能“打坏”。

## 典型场景与诊断步骤

### 场景1：控制台乱码
用 `Invoke-RestMethod` 获取 API 返回，直接在终端打印看到乱码。

诊断：
```powershell
# 查看当前代码页
chcp
# 查看 .NET 控制台输出编码
[Console]::OutputEncoding.EncodingName
```
常见返回是 `936`（GBK）或 `437`（OEM-United States）。此时控制台无法正确渲染 UTF-8 字节流。

快速修复：
```powershell
chcp 65001
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
```
然后重新执行 API 调用。

### 场景2：请求体中文丢失
发送包含中文的 JSON body，服务端收到全是 `?`。

根源在于 `Invoke-WebRequest` 默认使用 `$OutputEncoding` 将字符串转换为字节流。如果 `$OutputEncoding` 是 ASCII，所有非 ASCII 字符都会被替换为 `?`。

诊断：
```powershell
$OutputEncoding.EncodingName
# 可能显示 US-ASCII
```

修复：
```powershell
$OutputEncoding = [System.Text.Encoding]::UTF8
```
再执行请求。

### 场景3：文件输出乱码
你通过 `Out-File` 或 `>` 保存 JSON 到文件，然后用其他工具读取时发现乱码（比如 Node.js、Python、curl 直接读出错）。

诊断：用十六进制查看文件头。
```powershell
Format-Hex .\response.json -Count 4
```
如果文件头是 `FF FE`，说明是 UTF-16 LE BOM。

修复：
```powershell
# 使用 -Encoding 参数
$response | ConvertTo-Json -Depth 10 | Out-File -FilePath data.json -Encoding utf8
```
**注意：`-Encoding utf8` 在 PowerShell 5.x 中指的是 UTF-8 with BOM，PowerShell Core 6+ 才是无 BOM UTF-8。** Windows PowerShell 5.x 还是会有 BOM，但多数 JSON 解析器能容忍 BOM。如果接收端严格，需要再处理：
```powershell
$utf8NoBom = New-Object System.Text.UTF8Encoding $false
[System.IO.File]::WriteAllLines("data.json", $jsonString, $utf8NoBom)
```

## 踩坑点

1. **`$OutputEncoding` 和 `[Console]::OutputEncoding` 不是同一个东西**  
   `$OutputEncoding` 影响输出到管道、重定向和 Web 请求的编码；`[Console]::OutputEncoding` 影响控制台渲染。两者要分别设置。

2. **重定向运算符 `>` 编码固定为 UTF-16 LE**  
   `> output.json` 等同于 `Out-File -Encoding Unicode`。绝对不能用 `>` 保存 JSON 配置，除非你只在 PowerShell 之间交互。

3. **JSON 序列化的二次编码**  
   `ConvertTo-Json` 默认会转义非 ASCII 字符吗？PowerShell 5.x 的 `ConvertTo-Json` 带有 `-Compress` 时不会自动转义为 `\uXXXX`，但某些深层字符仍可能被误解。遇到顽固乱码，可显式指定 `-EscapeHandling EscapeNonAscii` 或通过 `[System.Text.Encoding]::UTF8.GetBytes()` 手动控制。

4. **curl.exe vs Invoke-RestMethod**  
   如果你在 PowerShell 里直接调 `curl.exe`（不是别名），编码受 curl 自身和 `chcp` 的影响，需要注意参数 `--data-binary` 和文件编码。

## 可复用建议

在你的工程化脚本或模块开头，建立一个统一的初始化块：

```powershell
# Windows PowerShell 5.x / PowerShell 7+ 通用编码初始化
if ($PSVersionTable.PSVersion.Major -lt 6) {
    # Windows PowerShell 环境
    $OutputEncoding = [System.Text.Encoding]::UTF8
    [Console]::OutputEncoding = [System.Text.Encoding]::UTF8
    $PSDefaultParameterValues['Out-File:Encoding'] = 'utf8'
    $PSDefaultParameterValues['*:Encoding'] = 'utf8'
    # 若需要无 BOM 输出，可自定义 Out-File 代理，但通常 BOM 可接受
} else {
    # PowerShell 7+ 默认编码已经是 UTF-8 无 BOM
    $OutputEncoding = [System.Text.Encoding]::UTF8
    [Console]::OutputEncoding = [System.Text.Encoding]::UTF8
}
# 统一设置控制台代码页
& chcp 65001 > $null
```

团队内部可以把这个 block 放到 profile 或模块的 `.psm1` 最前面，避免每人踩一遍坑。

对于发送 JSON 的场景，尽量显式传递 UTF-8 字节数组：

```powershell
$json = @{ message = "你好" } | ConvertTo-Json -Compress
$bodyBytes = [System.Text.Encoding]::UTF8.GetBytes($json)
Invoke-RestMethod -Uri https://api.example.com/echo -Method Post -Body $bodyBytes -ContentType "application/json; charset=utf-8"
```

这样完全绕开 `$OutputEncoding`，最彻底。

## 总结

PowerShell 在 Windows 上“打坏中文”并非玄学，而是以下三点叠加的结果：
- 控制台代码页不是 UTF-8
- `$OutputEncoding` 默认是 ASCII
- 文件输出默认使用 UTF-16 LE

了解这些默认行为后，只要在每个 API 调用的入口做好编码统一（chcp 65001 + `$OutputEncoding` + 显式字节化 body），就可以彻底根治。将这段初始化固化到团队脚本模板中，能极大降低跨工具联调时的摩擦，让 Windows 上的自动化 Agent、MCP 插件不再因为编码问题莫名其妙 500。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-28/c112570ab8f99ec5.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-28/5718335d4adfc32d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-28/3fd343abb25b7ee8.png)

