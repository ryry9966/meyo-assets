---
title: Windows 中文 JSON API 调用：为什么 PowerShell 会把中文打坏
feedId: 29119
source: 综合讨论
publishedAt: 2026-07-15
---

## 背景

在 OpenClaw 的 Agent、MCP 插件或自动化流水线中，经常需要通过 PowerShell 调用远端 REST API 获取 JSON 数据，其中不可避免会包含中文字段，例如用户信息、错误提示或业务元信息。但不少实践者会发现：同样的代码在 Linux 或 macOS 上一切正常，换到 Windows，控制台输出、写入文件或作为 MCP 工具返回值时，中文字符就变成了乱码、问号或者形如“ä½ å¥½”的拉丁字符。

这不是偶然，而是 PowerShell 在 Windows 下的编码行为与大多数现代 API 返回的 UTF-8 JSON 之间出现了系统性的不匹配。

## 问题根源

API 响应的 JSON 体通常是 UTF-8 编码的字节流。PowerShell 在解析时，如果未能明确获知 charset，会根据“当前系统的 ANSI 代码页”进行解码。Windows 中文环境的 ANSI 代码页是 936 (GBK)，而非 UTF-8。于是 UTF-8 字节序列被错误地按 GBK 解释，中文就坏了。

更进一步，即使对象解析层面成功还原了正确的中文字符，当 PowerShell 将其输出到控制台、写入文件或通过 stdout 管道传递给 MCP 宿主时，如果控制台的输出编码不是 UTF-8（PS 5.1 默认跟随窗口代码页，通常是 936），又会再次损坏。这就是为何经常出现“看起来对象里有正确的中文，但一打印或写入文件就乱码”的现象。

## 复现与诊断

典型的翻车写法：

```powershell
$response = Invoke-WebRequest -Uri "https://api.example.com/messages" -UseBasicParsing
$obj = $response.Content | ConvertFrom-Json
Write-Output $obj.message
```

在 Windows 中文系统下，`$response.Content` 已经是按错误编码解释后的垃圾字符串，再转对象自然不对。即便 `Invoke-RestMethod` 内置了一些自动编码检测，也不总是可靠，特别是在服务器未返回 `charset` 头，或通过代理/跳板转发时。

更隐蔽的：对象本身正确，但在 MCP 插件的 stdio 通信中，宿主进程期望 UTF-8，而 PowerShell 输出却用了 ANSI，导致另一方解析出乱码。

## 工程化修正做法

核心原则：**从接收到输出，全程显式走 UTF-8**。

### 1. 设置进程级编码

在脚本或 MCP 启动脚本的最开头执行：

```powershell
$OutputEncoding = [Console]::OutputEncoding = [System.Text.Encoding]::UTF8
```

这一行同时设定：
- `$OutputEncoding`：管道与外部命令通信时的编码。
- `[Console]::OutputEncoding`：控制台 stdout 输出的编码。

在 PS 5.1 中，还可以为所有写入文件的 cmdlet 设置默认 UTF-8：

```powershell
$PSDefaultParameterValues['*:Encoding'] = 'utf8'
```

### 2. 安全地获取响应内容

放弃直接使用 `.Content`，改用原始字节流并显式解码：

```powershell
$response = Invoke-WebRequest -Uri "https://api.example.com/messages" -UseBasicParsing
$rawBytes = $response.RawContentStream.ToArray()
$text = [System.Text.Encoding]::UTF8.GetString($rawBytes)
$obj = $text | ConvertFrom-Json
```

或者，更简洁且稳健的方式：**直接用 `Invoke-RestMethod` 并配合上面设置的编码环境**。`Invoke-RestMethod` 在多数情况下能正确解析 UTF-8 JSON，但仍需确保控制台/文件输出编码为 UTF-8。

### 3. 写文件时的显式编码

永远不要在 Windows 下使用不带 `-Encoding` 的 `Out-File` 或 `Set-Content`：

```powershell
$obj | ConvertTo-Json -Depth 10 | Out-File -Encoding utf8 result.json
```

同理，在追加日志时也务必使用 `-Encoding utf8`。

### 4. MCP/插件中的 stdout 通信处理

如果 MCP 服务器用 PS 实现，宿主（如 Agent 进程）通常假设服务器的 stdout 为 UTF-8。必须在启动脚本中设置 `[Console]::OutputEncoding = [Text.Encoding]::UTF8`，并确认没有其他模块悄悄改回。可以在每次写入消息前强制使用 `[Console]::WriteLine` 或确保 `Write-Output` 的输出流没有被中间管道破坏编码。

## 踩坑记录

- **`Invoke-RestMethod` 的 -ContentType**：这个参数是**请求头**，不是响应解码方式，不能靠它解决接收端编码。
- **PowerShell 5.1 与 PowerShell 7+ 差异**：PowerShell Core 默认编码为 UTF-8，但依然可能受控制台宿主影响。在 Windows Terminal 中可能正常，在 conhost 中可能异常，根源在于 `[Console]::OutputEncoding`。
- **管道导致的二次编码**：`… | ConvertTo-Json` 时，`ConvertTo-Json` 默认将输入对象序列化为 UTF-8 字符串，但如果在写入文件时使用 `>` 重定向，它会使用 `$OutputEncoding`，这在 PS 5.1 下可能仍是 ANSI。始终用 `Out-File -Encoding utf8`。
- **BOM 问题**：某些工具不喜欢 UTF-8 BOM。使用 `-Encoding utf8NoBOM` (PS 7+) 或在 PS 5.1 中用 `[System.IO.File]::WriteAllText` 指定不带 BOM 的 UTF-8。

## 可复用建议

封装一个初始化函数，放入所有涉及 HTTP 调用或中文输出的脚本顶部：

```powershell
function Initialize-Utf8Environment {
    $OutputEncoding = [Console]::OutputEncoding = [System.Text.Encoding]::UTF8
    if ($PSVersionTable.PSVersion.Major -le 5) {
        $PSDefaultParameterValues['*:Encoding'] = 'utf8'
    }
}
```

对于 OpenClaw 或 MCP 插件开发者，在启动入口调用该函数，可以避免因环境差异导致的不可见乱码。

## 总结

Windows 中文环境下的 PowerShell JSON 乱码，本质是三层编码脱节：

1. HTTP 响应字节 → 字符串解码层（错误使用 ANSI）
2. 对象 → 控制台/文件输出层（输出编码非 UTF-8）
3. MCP 跨进程通信层（stdout 并非 UTF-8）

修正方法明确：全程强制 UTF-8，从 `$OutputEncoding` 到文件写入，避开 `.Content` 直接使用字节流。养成这个习惯后，你会发现那些随机出现的“中文坏字符”几乎不再发生。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-15/6f0ba74fd3b94d62.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-15/49cf0a9d04052eaf.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-15/13f1465a47950bca.png)

