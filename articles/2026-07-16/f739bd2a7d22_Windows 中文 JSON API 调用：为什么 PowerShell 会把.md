---
title: Windows 中文 JSON API 调用：为什么 PowerShell 会把中文打坏
feedId: 29233
source: 综合讨论
publishedAt: 2026-07-16
---

## 背景
在 OpenClaw、MCP Server 或本地自动化插件的开发中，Windows 上的 PowerShell 经常被用作 HTTP 客户端：通过 `Invoke-RestMethod` 或 `Invoke-WebRequest` 调用中文 API，期望返回可读的 JSON。但工程师几乎都踩过同一个坑——返回值里的中文在控制台显示为 `??`、`锟斤拷` 或不可解析的乱码，甚至在 `ConvertFrom-Json` 后就彻底损坏。表面看是“中文乱码”，实际根因是一整条编码链上的不一致。

## 问题定位：编码断层发生在哪里？
一个典型的调用链：

```powershell
$response = Invoke-RestMethod -Uri "https://api.example.com/data"
Write-Output $response.name   # 输出 ?? 或乱码
```

很多人会尝试这样修复：
```powershell
$response = Invoke-RestMethod -Uri "https://api.example.com/data" -ContentType "application/json; charset=utf-8"
```
但问题依旧。原因在于：**API 返回的原始字节流本来就是合法的 UTF-8，问题出在 PowerShell 接收到数据后的处理与显示环节。**

具体断点：
1. **控制台输出编码**  
   Windows 控制台（conhost 或 Windows Terminal 的国内默认配置）的 `[Console]::OutputEncoding` 通常是系统代码页（如 GBK/936）。当 PowerShell 将内部 Unicode 字符串写回控制台时，会按此编码进行转换，UTF-8 中 3 字节的中文字符无法映射到 GBK 宽字符集，导致丢失或变成 `?`。
2. **重定向与文件输出编码**  
   使用 `>` 或 `Out-File` 时，Windows PowerShell 5.1 的默认写入编码是 UTF-16 LE（`Unicode`），而多数外部工具和编辑器期望的是 UTF-8。这会导致 `>>` 追加乱码、跨工具链断裂。
3. **`Invoke-RestMethod` 内部解码**  
   `Invoke-RestMethod` 会根据响应头 `Content-Type` 的 charset 自动解码为 .NET 字符串。如果服务端使用了 `application/json` 但没有 `charset`，某些 Windows HTTP 栈会回退到 ISO-8859-1 或系统代码页，而非 RFC 建议的 UTF-8，这会将原始 UTF-8 字节错误地解码一次。不过这种情况相对少见，更常见的问题出在前两项。

## 可复用的修复步骤
### 1. 为当前会话修复控制台编码
在脚本开头或 profile 中设置：
```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$OutputEncoding = [System.Text.Encoding]::UTF8
```
说明：
- `[Console]::OutputEncoding` 影响直接写控制台的显示。
- `$OutputEncoding` 影响管道到外部命令（如 `findstr`）时的编码。但注意，它**不会**影响 `Out-File` 或 `Set-Content` 的默认编码。

### 2. 使用 `Invoke-WebRequest` 获取原始字节后手工解码
最稳健的方式是绕过自动解码：
```powershell
$response = Invoke-WebRequest -Uri "https://api.example.com/data" -Method Get
$utf8String = [System.Text.Encoding]::UTF8.GetString($response.RawContentStream.ToArray())
$data = $utf8String | ConvertFrom-Json
```
这里的 `RawContentStream` 拿到的是原始字节，可以精确控制解码方式。注意需要 PowerShell 5.1 及以上，并且优先使用 `-UseBasicParsing` 避免 Internet Explorer 解析器干扰（Windows Server 核心环境推荐）。

### 3. 正确写出到文件
所有写入文件的 cmdlet 显式指定 `-Encoding UTF8`：
```powershell
$data | ConvertTo-Json -Depth 10 | Out-File -FilePath "result.json" -Encoding UTF8
```
避免使用 `>>`，因为它的编码行为取决于 PowerShell 版本且不具备可控性。

### 4. 切换到 PowerShell 7 (pwsh) 作为默认运行环境
PowerShell 7 做了重要的默认值调整：
- `Out-File` 和 `Set-Content` 默认编码改为 **UTF8NoBOM**。
- `$OutputEncoding` 默认为 UTF-8。
- 跨平台一致性更好，尤其是为 MCP / Agent 编写跨操作系统脚本时。

如果还停留在 5.1，强烈建议在自动化入口通过 `pwsh` 调用脚本。

## 踩坑点记录
- **UTF8 与 UTF8NoBOM**  
  某些 Windows 工具（如旧版记事本）只识别带 BOM 的 UTF-8。如果脚本需要被这些工具消费，可改用 `-Encoding UTF8BOM`。
- **Windows Terminal vs conhost**  
  Windows Terminal 即使系统代码页为 936，也可以正确显示 UTF-8 多字节字符，而 conhost 会失败。不要因为 Terminal 上看似正常就认为编码绝对正确；输出重定向到文件后的检验才是真标准。
- **`Invoke-RestMethod` 返回对象时的显示假象**  
  返回的 `PSCustomObject` 在存储时已经是正确的 .NET 字符串，只是 `Write-Output` 时转换错误。许多人误认为内容已损坏，而实际上对象内部的字符串仍然完好。可以通过检查 `[System.Text.Encoding]::UTF8.GetBytes($response.name)` 来确认。

## 可复用建议：封装安全请求函数
在 OpenClaw 或 MCP 工具链中，直接暴露裸 `Invoke-RestMethod` 并不安全。可封装如下：

```powershell
function Invoke-SafeRestMethod {
    param($Uri, $Method = 'Get')
    $req = Invoke-WebRequest -Uri $Uri -Method $Method -UseBasicParsing
    $raw = $req.RawContentStream.ToArray()
    $json = [System.Text.Encoding]::UTF8.GetString($raw)
    return $json | ConvertFrom-Json
}
```

如果需要保持 API 响应的二进制保真度（例如传递给另一个服务），直接使用字节数组并设置正确的响应头转发，切勿将字节流转换成字符串再转回，那样必然引入二次编码风险。

对于基于管道流的 MCP 插件，在启动脚本中统一声明编码设置：
```powershell
[Console]::OutputEncoding = [Console]::InputEncoding = [Text.Encoding]::UTF8
$OutputEncoding = [Text.Encoding]::UTF8
$PSDefaultParameterValues['*:Encoding'] = 'utf8'
```
这可以堵住大部分因为 PowerShell 隐式编码导致的 API 数据损坏。

## 总结
PowerShell 在 Windows 下处理中文 JSON 的乱码，本质不是网络传输的问题，而是字符串从 .NET 内部码点转换到控制台或文件系统时的编码不匹配。理解三个关键编码变量（控制台输出、管道输出、文件输出）并显式控制它们，是 Windows 自动化开发必须养成的基础习惯。在 OpenClaw 这类以 API 管道为核心的工程中，采用 `RawContentStream` 手工解码 + PowerShell 7 默认 UTF-8 + 统一参数化编码，可以从根源上把中文“打不坏”。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-16/d5e76f26af0a85cc.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-16/6d5d26cacfa0d950.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-16/5437d248a0770901.png)

