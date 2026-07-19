---
title: Windows PowerShell 调用中文 JSON API：编码陷阱与工程化修复实践
feedId: 29623
source: 综合讨论
publishedAt: 2026-07-19
---

## 背景
在 OpenClaw、Agent 插件、MCP 工具以及各类自动化流水线里，调用 REST API 并处理中文 JSON 是家常便饭。Windows 用户习惯用 PowerShell 快速发请求、解析返回数据，但是不少工程师第一次在返回值里看到“?????”或“涓€涓?”时，都会怀疑 API 本身出了问题——实际上，绝大多数情况是 PowerShell 的编码机制把中文字符打坏了。

本文从工程视角梳理问题根因，给出在 Windows PowerShell 与 PowerShell Core 下可稳定复用的解决步骤，避免你在 Agent 自动化中反复踩同一个坑。

## 问题表现
两个典型场景：
1. **发送 JSON 请求体**：使用 `Invoke-RestMethod` 提交含中文的 JSON Body，服务端收到的却是乱码或无效的 UTF-8 字节流。
2. **接收 API 响应**：返回的 JSON 中有中文，控制台输出变成一堆问号（`??`），或者用 `Out-File` 写入文件后，文件里的中文变成 `宸ヤ綔` 之类的乱码组合。

更麻烦的是，同样的脚本在 PowerShell Core（pwsh 7+）里可能完全正常，只在 Windows PowerShell 5.1 中出现——这让很多新人误以为“windows 就是搞不定中文”。

## 根因分析
Windows PowerShell 5.1 在字符编码上有三处关键默认值，专门坑中文：

- **$OutputEncoding**：控制 `>` 重定向以及外部命令（如 `curl`，其实是 `curl.exe`）输出的默认编码。Windows PowerShell 默认是 `ASCIIEncoding`，仅支持 0–127 的字符，所有非 ASCII 字符会被强制转换为 `?`。
- **管道/文件输出的默认编码**：`Out-File`、`Set-Content` 默认使用 `Unicode`(UTF-16LE) 或 `ASCII`（取决于命令），不一定是你期望的 UTF-8。`Invoke-WebRequest` 的 `.Content` 属性在内部解码时，如果没有显式指定 charset，可能回退到错误的代码页。
- **控制台显示编码**：`[Console]::OutputEncoding` 决定终端窗口用什么编码显示文本，即使字符串对象内部是正确的中文，显示也会因控制台编码不符而变成乱码。但这只是显示问题，不影响字符串值本身。

当你用 `Invoke-RestMethod` 发 JSON 时，`-Body` 参数如果是字符串，则 PowerShell 会将该字符串转换成字节流发送。如果这个字符串本身就是用错误的编码生成的（比如从 `Get-Content` 用默认 ASCII 读取后拼接的），那么请求体就已经损坏。接收响应时，如果服务端返回头里没有明确 `charset=utf-8`，PowerShell 可能用 `ISO-8859-1` 去解码，亦导致中文字符丢失。

## 解决步骤（可复现）
### 1. 优先切换到 PowerShell Core
在 pwsh 7+ 中，`$OutputEncoding` 默认为 UTF-8，`Out-File` 和网络组件也更符合现代 Web 编码习惯。如果你能控制执行环境，**这是最彻底的解决办法**。

### 2. Windows PowerShell 下的硬修复
若环境必须是 Windows PowerShell（如某些企业管控），请在执行脚本头部统一设置：
```
# 修正输出编码
$OutputEncoding = [System.Text.Encoding]::UTF8
# 使所有外部命令和重定向使用 UTF-8
$PSDefaultParameterValues['*:Encoding'] = 'utf8'
# 尝试修改控制台显示编码（需要终端字体支持）
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
```

### 3. 构造请求体必须保证字节级正确
不要直接用拼接字符串生成 JSON Body。使用 `ConvertTo-Json` 且确保 `-Depth` 足够，再转为字节数组：
```powershell
$bodyObj = @{ name = "测试计划"; enabled = $true }
$jsonBody = $bodyObj | ConvertTo-Json -Depth 3
$bodyBytes = [System.Text.Encoding]::UTF8.GetBytes($jsonBody)
```
然后通过 `Invoke-RestMethod` 发送：
```powershell
$response = Invoke-RestMethod -Uri $apiUrl -Method Post `
    -ContentType "application/json; charset=utf-8" `
    -Body $bodyBytes
```
这里 `-Body` 传入字节数组时，PowerShell 不会再次转换编码，直接使用原始字节。

### 4. 接收响应时强制 UTF-8 解码
如果 API 返回头里没有 charset，可以用 `Invoke-WebRequest` 获取原始字节后手工解码：
```powershell
$resp = Invoke-WebRequest -Uri $apiUrl `
    -ContentType "application/json; charset=utf-8"
if ($resp.Content -match '[\u4e00-\u9fff]' -and $resp.RawContentStream) {
    $streamReader = [System.IO.StreamReader]::new($resp.RawContentStream, [System.Text.Encoding]::UTF8)
    $content = $streamReader.ReadToEnd()
    $streamReader.Close()
    $json = $content | ConvertFrom-Json
} else {
    $json = $resp.Content | ConvertFrom-Json
}
```
如果仅用 `Invoke-RestMethod`，它通常能正确解析 JSON 对象，中文不会被破坏，但前提是返回头声明了 UTF-8。若无声明，可考虑在 API 网关层统一加上。

### 5. 持久化结果到文件
任何文件写入都必须显式指定 UTF-8：
```powershell
$jsonOutput | Out-File -FilePath "result.json" -Encoding utf8
```
或通过 `Set-Content`：
```powershell
Set-Content -Path "result.json" -Value $jsonOutput -Encoding utf8
```

## 踩坑点总结
- **$PSDefaultParameterValues 只对 cmdlet 有效**，无法修复外部命令如 `curl.exe` 的输出，必须同时设置 `$OutputEncoding`。
- **控制台字体不包含中文字符集时，即使编码正确也会显示乱码**，切换到 `Consolas` 或 `SimHei` 等支持中文的字体。
- **`ConvertTo-Json` 默认 `-Depth 2`**，复杂对象会导致截断，务必根据实际层级显式加大。
- **`Invoke-RestMethod` 的 `-Body` 传入 `[hashtable]` 时，会自动序列化为 JSON 并编码成 UTF-8**，这通常不会引起中文问题，前提是你没有提前进行编码干预。
- **如果使用 `curl` 别名（指向 `Invoke-WebRequest`）**，要注意参数差异，建议统一使用完整 cmdlet 名称提高可读性。

## 可复用建议
在你的 OpenClaw 或 Agent 项目的 PowerShell 模块里，封装一个初始化函数：
```powershell
function Initialize-Encoding {
    if ($PSVersionTable.PSVersion.Major -lt 6) {
        $OutputEncoding = [System.Text.Encoding]::UTF8
        $global:PSDefaultParameterValues['*:Encoding'] = 'utf8'
        [Console]::OutputEncoding = [System.Text.Encoding]::UTF8
    }
}
```
同时在封装 HTTP 调用时内置“字节发送 + 字节接收 + UTF-8 保证”，避免每次手写相同逻辑。

对于基于 MCP 的工具，若 server 端返回的中文消息在 Windows 客户端显示异常，通常也是 stdio/stdout 编码未同步，可同样使用 `$OutputEncoding` 修复。

## 总结
Windows 上 PowerShell 的中文 JSON 乱码不是玄学，而是几处编码默认值落后于 UTF-8 时代的历史遗留问题。通过系统化设置 `$OutputEncoding`、手动控制字节流、以及优先选用 PowerShell Core，你的 Agent 自动化流水线完全可以可靠地处理中文 API 交互。遇到“????”不用慌，查编码设置比怀疑 API 更有效。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-19/39b21ebb1d3a7c73.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-19/ec5e3e39c6018d47.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-19/8e404b86567a2d34.png)

