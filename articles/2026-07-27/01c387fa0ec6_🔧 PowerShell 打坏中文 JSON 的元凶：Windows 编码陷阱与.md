---
title: 🔧 PowerShell 打坏中文 JSON 的元凶：Windows 编码陷阱与一套根治方案
feedId: 30683
source: 综合讨论
publishedAt: 2026-07-27
---

## 背景：自动化脚本里的“幽灵乱码”

在给 OpenClaw 搭 MCP 插件、Agent 工具链或 API 调度脚本时，Windows 上的 PowerShell 是绕不开的执行器。只要你的 JSON 里出现中文——无论是请求体还是文件名——十有八九会遇上这类场面：

- API 收到的 `"name":"测试"` 变成 `"name":"\u6d4b\u8bd5"`
- 文件内容保存后 `你好` 变成 `ä½ å¥½`
- 输出到控制台的中文全显示 `???`

这些不是“玄学”，是**字符集（charset）在 PowerShell 管线的每一层都可能被错误编码/解码**。本文面向在 Windows 上用 PowerShell 调中文 API 的工程化实践，给出可复现的原因和可复用的修正路径。

## 问题在手：重现“打坏”的全过程

先看一段最朴素的调用脚本，在 Windows PowerShell 5.1 里执行：

```powershell
$body = @{ title = "中文标题" } | ConvertTo-Json
Invoke-RestMethod -Uri "https://httpbin.org/post" -Method Post -Body $body -ContentType "application/json; charset=utf-8"
```

**现象 1**：`$body` 直接被打印时可能正常，但一旦重定向到文件或作为 API 请求体，中文变成了 `\u4e2d\u6587\u6807\u9898`。  
**现象 2**：API 返回的中文在控制台显示乱码，即使对方明确返回 UTF-8。  
**现象 3**：如果不用 `Invoke-RestMethod`，改用 `curl.exe`，又一切正常。

根本原因在于三层编码错位：  
1. **`ConvertTo-Json` 的默认转义逻辑**  
2. **PowerShell 输出编码（`$OutputEncoding`）与管线传递的编码不一致**  
3. **Console 宿主对多字节文字的渲染能力差异**（旧控制台 vs Windows Terminal）

## 层级拆解：每一环都在破坏中文

### 1. `ConvertTo-Json`：你用的是什么 PS 版本？

Windows PowerShell 5.1 自带的 `ConvertTo-Json` **没有** `-EscapeHandling` 参数。它对所有非 ASCII 字符，默认转义为 `\uXXXX` 形式。这是合法的 JSON（RFC 8259），可读性极差，但 API 通常能正确解码——前提是后续编码传输不搞鬼。  
如果你用 PowerShell 7+（PowerShell Core），可以直接：

```powershell
$body = @{ title = "中文标题" } | ConvertTo-Json -EscapeHandling EscapeHtml
```

这样会输出 `"title":"中文标题"` 而非转义码。但对于混合中英文字段，`EscapeNonAscii` 才是控制非 ASCII 转义与否的参数。  
**不能升级 PowerShell 的场景**，用 Newtonsoft.Json 迂回：

```powershell
Add-Type -Path "Newtonsoft.Json.dll"
$body = [Newtonsoft.Json.JsonConvert]::SerializeObject(@{ title = "中文标题" })
```

### 2. 管线与输出编码：`$OutputEncoding` 的软刀子

当用 `Invoke-RestMethod` 发送字符串 body 时，PowerShell 会将 `$body` 转换为字节流。此时它会按照 `$OutputEncoding` 决定用什么字符集去编码这个字符串。  
**Windows PowerShell 的默认 `$OutputEncoding` 是 ASCII !** 所以即使 `$body` 里包含正确的 UTF-8 元数据，发送时也会被截断成 `?`。  
标准修复：

```powershell
$OutputEncoding = [System.Text.Encoding]::UTF8
$body = @{ title = "中文标题" } | ConvertTo-Json
Invoke-RestMethod -Uri "..." -Body $body -ContentType "application/json; charset=utf-8"
```

如果你的脚本里混合了外部命令（如 `curl`），最好连同 `[Console]::OutputEncoding` 也一并设成 UTF-8。

### 3. 持久化存储：`Out-File` 的 BOM 陷阱

将中文 JSON 写入文件时，很多人随手用 `Out-File` 或 `>`。默认编码是 UTF-16 LE 或带 BOM 的 UTF-8（视主机而定）。一些处理 UTF-8 的 API 或工具会被 BOM 卡死。  
安全写法：

```powershell
$json | Out-File -Encoding utf8NoBOM data.json
```

或直接用 .NET：

```powershell
[System.IO.File]::WriteAllText("data.json", $json, [System.Text.UTF8Encoding]::new($false))
```

### 4. 控制台渲染：Windows Terminal 是底线

旧版 CMD/PowerShell Console 无法良好渲染 CJK 文字，即便编码已正确，显示依然是问号。这不是脚本问题，是终端问题。强烈建议在 Windows Terminal 环境中运行所有自动化脚本，并在终端设置里把“Use Unicode UTF-8 for worldwide language support”打开（勾选 Beta 选项）。

## 一套可复用模板

下面模板兼顾 PowerShell 5.1 和 7+，专为中文 JSON API 调用设计：

```powershell
# 全局编码锁定 UTF-8
$OutputEncoding = [Console]::OutputEncoding = [System.Text.Encoding]::UTF8

# 构造 JSON（根据 PS 版本选择转义策略）
if ($PSVersionTable.PSVersion.Major -ge 7) {
    $body = @{ title = "中文标题" } | ConvertTo-Json -Compress -EscapeHandling EscapeNonAscii
} else {
    # PS5.1 回退方案
    $body = [Newtonsoft.Json.JsonConvert]::SerializeObject(@{ title = "中文标题" })
}

# 显式传递 UTF-8 字节（最稳妥）
$bytes = [System.Text.Encoding]::UTF8.GetBytes($body)
$response = Invoke-RestMethod -Uri "https://httpbin.org/post" -Method Post -Body $bytes -ContentType "application/json; charset=utf-8"
```

关键点：**直接传递字节数组** 可以跳过 `$OutputEncoding` 的黑箱，也是最不容易出错的方式。

## 踩坑清单

- **PowerShell 5.1 的 `-Body` 传字符串会被二次编码**：上面已经用字节数组规避。  
- **`Invoke-WebRequest` 的 `-OutFile` 保存文件时默认写入 UTF-16 LE**，解决方法一样是用 .NET 手动写文件。  
- **外部命令 `curl.exe` 调用时如果 shell 是 Powershell，传参中的中文可能被解析错误**，建议全部参数用 `--data-binary @file.json` 读取 UTF-8 文件，避开命令行转义。  
- **某些 API 网关或负载均衡会把 `\uXXXX` 字符串当作普通字符**，导致业务侧无法正确解析中文。此时必须让 PowerShell 输出原生中文 JSON，即强制不转义。

## 总结

PowerShell 打坏中文 JSON，表面是乱码，本质是**编码传递链的每一次转换都可能引入错误**。工程上的收敛方案就是：  
1. 强制 UTF-8 无处不在（`$OutputEncoding`、`[Console]::OutputEncoding`、文件写入编码）。  
2. 尽量升级到 PowerShell 7+，获取更可控的 JSON 转义参数。  
3. 对 `Invoke-RestMethod` 直接传 UTF-8 字节数组，终结猜测。  
4. 始终在支持 Unicode 的现代终端里运行脚本。

当你下次发现 `\u4e2d` 冒出来时，就不会再去改 server 端的接收逻辑了——问题永远在发送方。

---

