---
title: Windows 中文 JSON API 调用：为什么 PowerShell 会把中文打坏
feedId: 28883
source: 综合讨论
publishedAt: 2026-07-13
---

在 OpenClaw 以及相关 MCP 插件的自动化实践中，Windows 环境下的 Agent 经常需要通过脚本调用本地或远端 API。当我们使用 PowerShell 编写胶水代码处理包含中文的 JSON 数据时，经常会遇到一个令人头疼的问题：中文字符被打坏，变成 `????` 或乱码，导致 JSON 解析失败，Agent 流程中断。

这并非 API 端的 Bug，而是 PowerShell 在 Windows 上的历史编码包袱。本文将梳理问题根源，并给出工程化的解决步骤。

### 问题背景

在 Windows 环境下，默认的 PowerShell（特指 v5.1）控制台编码通常是 GBK（代码页 936）或系统默认的 ANSI 编码。而现代 JSON API 标准均要求使用 UTF-8。

当我们在 PowerShell 中构造一个包含中文的哈希表，将其转为 JSON 字符串，并通过 `Invoke-RestMethod` 发送时，PowerShell 会按照当前控制台的默认编码（而非 UTF-8）将字符串转换为字节流发送出去。API 端按 UTF-8 解析，自然无法正确识别，导致中文损坏。

### 解决做法与步骤

要彻底解决此问题，需要在发送请求和接收响应两个环节显式接管编码处理。

**1. 统一控制台与默认编码**
在脚本开头强制将当前会话的输入输出编码设为 UTF-8：
```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$OutputEncoding = [System.Text.Encoding]::UTF8
```

**2. 手动序列化并指定 UTF-8 发送**
不要直接将对象交给 `Invoke-RestMethod` 的 `-Body` 参数，而是先手动转换为 JSON，并明确使用 UTF-8 获取字节数组：
```powershell
$headers = @{
    "Content-Type" = "application/json; charset=utf-8"
}

$payload = @{
    query = "查询 OpenClaw 节点状态"
    lang  = "zh-CN"
} | ConvertTo-Json -Depth 10

# 核心：按 UTF-8 编码转换为字节流
$bodyBytes = [System.Text.Encoding]::UTF8.GetBytes($payload)

Invoke-RestMethod -Uri "https://api.example.com/v1/agent" `
    -Method Post `
    -Headers $headers `
    -Body $bodyBytes
```

**3. 正确处理 API 响应中的中文**
如果 API 返回的 JSON 中包含中文，且响应头未明确声明 `charset=utf-8`，PowerShell 5.1 可能会按默认编码解析。更稳妥的做法是使用 `Invoke-WebRequest` 获取原始字节，再手动按 UTF-8 解码：
```powershell
$response = Invoke-WebRequest -Uri "https://api.example.com/v1/data" -Method Get
# 强制按 UTF-8 解码响应体
$responseText = [System.Text.Encoding]::UTF8.GetString($response.RawContentStream.ToArray())
$data = $responseText | ConvertFrom-Json
```

### 踩坑点

1. **PowerShell 5.1 与 7.x 的差异**：如果你在 Windows 上使用 PowerShell 7（Core），默认编码已经是 UTF-8，上述问题可能不会复现。但在自动化部署或调用本地 MCP 插件时，系统默认调用的往往是 `powershell.exe`（5.1），而非 `pwsh.exe`（7.x），切勿混淆环境。
2. **`ConvertTo-Json` 的转义问题**：在 Windows PowerShell 5.1 中，`ConvertTo-Json` 会将非 ASCII 字符转义为 `\uXXXX` 形式。虽然这能保证中文不被打坏，但会导致 JSON 体积膨胀。如果需要明文中文，需结合 `[System.Text.Encoding]::UTF8` 处理。
3. **BOM 头引发的血案**：使用 `Out-File` 或 `Set-Content` 保存 JSON 文件时，PowerShell 5.1 默认会写入 UTF-8 BOM。部分严格的 JSON 解析器（包括某些 Agent 框架）遇到 BOM 会报错。保存时应使用 `New-Item` 配合 `[System.IO.StreamWriter]`，或指定 `-Encoding utf8NoBOM`（仅限 PS 7+）。

### 可复用建议

- **封装统一请求函数**：在 OpenClaw 插件或 Agent 脚本库中，封装一个 `Invoke-JsonApi` 函数，内部固化 UTF-8 字节转换逻辑，屏蔽底层差异。
- **环境检测与降级**：在脚本入口检测 `$PSVersionTable.PSVersion.Major`。如果是 7 以下版本，主动注入 UTF-8 编码设置；有条件的话，直接要求环境安装 PowerShell 7。
- **避免隐式类型转换**：处理 JSON 时，尽量在字节流（`byte[]`）层面与 HTTP 交互，避免 PowerShell 字符串的隐式编码干预。

### 总结

Windows PowerShell 处理中文 JSON 乱码的根源在于默认编码与现代 Web 标准的错位。在 Agent 与 MCP 插件的工程实践中，我们不能依赖运行时的隐式转换，必须在序列化、HTTP 请求、反序列化三个环节显式声明 UTF-8 编码。通过字节级别的精准控制，才能确保中文数据在自动化链路中无损流转。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-13/7608acf571f10f24.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-13/526ded1c877fdb98.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-13/0d20edda93bce93f.png)

