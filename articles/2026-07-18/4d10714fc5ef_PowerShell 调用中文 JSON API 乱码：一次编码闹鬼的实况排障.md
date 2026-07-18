---
title: PowerShell 调用中文 JSON API 乱码：一次编码闹鬼的实况排障
feedId: 29492
source: 综合讨论
publishedAt: 2026-07-18
---

## 背景

在 Windows 上用 PowerShell 写自动化脚本调用 HTTP API，已经是 Agent、MCP 工具、OpenClaw 插件的日常操作。不少 API 返回的 JSON 里包含中文字段——城市名、状态描述、用户昵称，甚至是 LLM 流式输出的部分结果。然而脚本在服务器上跑得好好的，一拿到某些 Windows 机器上，`Invoke-RestMethod` 或 `Invoke-WebRequest` 拿回来的中文就变成了一堆问号、锟斤拷，或者形如 `ä½ å¥½` 的乱码。

更让人头疼的是，同样的代码在 PowerShell 控制台直接运行，偶尔显示正常；一旦把结果用 `Out-File` 写入文本，或者在后续的 `ConvertFrom-Json` 里取值再拼装，中文就被打坏。对于依赖 JSON 字段做判断的 Agent，这就意味着动作可能被误导，比如把“运行中”识别成乱码后走了错误分支。

## 问题根源

PowerShell 并非天生不懂 UTF-8，但它和 Windows 控制台、.NET 运行时的历史包袱交织在一起，产生了几层叠加的编码陷阱。

1. **响应内容在内存里未必就是 UTF-8**  
   `Invoke-RestMethod` 内部使用 `HttpClient`，会根据服务器返回的 `Content-Type` 头部的 `charset` 自动解码字节流。如果 API 没有明确声明 `charset=utf-8`，而是返回 `Content-Type: application/json`，甚至返回 `text/plain`，那么 .NET 可能退回到 `ISO-8859-1` 或者系统代码页（简体中文 Windows 通常是 GBK/CP936）。中文字节在这些编码下被错误解释，内存里的字符串就已经损坏。

2. **控制台输出编码与终端的差距**  
   PowerShell 5.1 的 Windows 控制台默认输出编码是 `Windows-1252`（即使系统 UI 代码页是 936）。这意味着即便内存中的字符串正确，当它被渲染到控制台时，如果字符串包含 GBK 范围之外的字符，也可能被转成 `?`。在 PowerShell 7+ 里情况好转，因为它预期终端支持 UTF-8，但如果终端字体不支持或者代码页没切到 `65001`，依然会出现显示问题。

3. **重定向、管道与文件输出的编码断裂**  
   最典型的“打坏”发生在 `>`, `Out-File`, `Set-Content` 等操作。PowerShell 5.1 默认使用 `Unicode`（UTF-16 LE）作为文件输出编码，这听起来无害，但如果你在 `>` 重定向时没有显式指定编码，它会使用 ANSI 代码页输出——也就是 GBK。而如果你的字符串在内存里已经是劣化的 GBK 表示，写出去就再无恢复可能。甚至是 `ConvertTo-Json` 本身：在 Windows PowerShell 5.1 中，它默认会用 `ASCII` 编码转义非 ASCII 字符，把“中”变成 `\u4e2d`，这在 API 里可能没问题，但在日志文件里可读性全无。

## 再现与固化的实验步骤

假设你要调用一个查询城市天气的 API，返回 `{"city":"北京","status":"晴天"}`。

**1. 先构建最小复现环境**（以 Windows PowerShell 5.1 为例）  
```powershell
$response = Invoke-RestMethod -Uri "https://api.example.com/weather?city=beijing"
$response.city
```
控制台输出是一堆乱码或问号。

**2. 检查原始字节**  
确认问题在于解码而非显示：
```powershell
$raw = Invoke-WebRequest -Uri "https://api.example.com/weather?city=beijing"
$raw.RawContentStream.Position = 0
$reader = [System.IO.StreamReader]::new($raw.RawContentStream, [System.Text.Encoding]::UTF8)
$rawString = $reader.ReadToEnd()
$rawString
```
如果此时看到正常中文，可以断定是 `Invoke-RestMethod` 自动解码时选错了编码。

**3. 强制使用 UTF-8 解码**  
如果 API 不肯加 charset，或者你没办法控制服务端，可以手动指定编码：
```powershell
$raw = Invoke-WebRequest -Uri "https://api.example.com/weather?city=beijing"
$utf8 = [System.Text.Encoding]::UTF8
$decoded = $utf8.GetString($raw.Content)
$json = $decoded | ConvertFrom-Json
$json.city
```
这样做跳过了自动 charset 检测，强制按 UTF-8 解码。

**4. 安全地输出到文件**  
即使内存里字符正确，到了写文件环节也要小心：
```powershell
$json.city | Out-File -FilePath "city.txt" -Encoding utf8
```
不要用 `>`，因为它在某些宿主里等同于 `Out-File -Encoding Default`（即 ANSI）。

**5. 彻底统一会话的编码设置**  
对于一批脚本，可以在脚本开头设定：
```powershell
$OutputEncoding = [System.Text.Encoding]::UTF8
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
```
`$OutputEncoding` 影响管道传给外部程序时的编码；`[Console]::OutputEncoding` 则影响控制台显示。两者都设成 UTF-8，能解决大部分的输入输出扭曲。

如果还不能根除，就在 profile 里加入 `chcp 65001`，或强制启动 PowerShell 7。

## 踩坑点

- **BOM 问题**：`Out-File -Encoding utf8` 会产生带 BOM 的 UTF-8 文件，一些 API 客户端或者 `cat` 命令可能会把 BOM 当作文本的一部分。如果需要无 BOM，可以使用 `[System.IO.File]::WriteAllText("path", $string, [System.Text.UTF8Encoding]::new($false))`。
- **转换链路中的多次编解码**：从 `Invoke-WebRequest` 获取 `Content` 属性时，PowerShell 已经将其解码为字符串，如果原始字节被错误解码，这个字符串就已损坏。最稳妥的方式是永远从 `RawContentStream` 或 `Content` 的字节形式入手，再显式指定编码重建字符串。
- **ConvertFrom-Json 的 `-AsHashtable` 与深度嵌套**：在 PowerShell 7 里，用 `ConvertFrom-Json -AsHashtable` 处理 JSON 更符合后端逻辑，但不影响编码问题本身。乱码的字符串无论放在自定义对象还是哈希表里都是损坏的。
- **远程会话和 CI 环境**：在 SSH 远程会话里，终端的 `LANG` 环境变量与实际编码可能不统一，导致 `chcp` 调整失效。此时应优先使用 PowerShell Core，它内部已经深度拥抱 UTF-8。

## 可复用的工程建议

1. **约定 API 返回 `Content-Type: application/json; charset=utf-8`**。如果你是 API 提供方，加上这个头是最彻底的解法。如果你是调用方，尽量在文档里推动上游修正。
2. **在脚本模板头部加入编码护盾**：  
   ```powershell
   $PSDefaultParameterValues['*:Encoding'] = 'utf8'
   $OutputEncoding = [Console]::OutputEncoding = [System.Text.Encoding]::UTF8
   ```
3. **优先使用内存安全的模式**：  
   ```powershell
   $response = Invoke-WebRequest -Uri $uri
   $contentBytes = $response.RawContentStream.ToArray()
   $string = [System.Text.Encoding]::UTF8.GetString($contentBytes)
   $data = $string | ConvertFrom-Json
   ```
   把解码权牢牢握在自己手里。
4. **在 MCP 工具或 Agent 插件的日志落盘时**，始终用无 BOM UTF-8。确保日志分析工具（如 grep、jq）不会因 BOM 而解析失败。
5. **构建自动化测试**：在 CI 中显式检查返回中文 JSON 的字段是否可读，而不是只验证 HTTP 状态码。

## 总结

PowerShell 打坏中文 JSON，本质上不是 Windows 的“原罪”，而是从 Win32 控制台时代遗留下来的编码假设与 UTF-8 现代网络之间的摩擦。对于 OpenClaw 生态里的 Agent 和插件开发者，这一点尤其致命——一个错码的“等待”字段可能让工作流永远卡死。好在解决手段也足够明确：夺回解码控制权，统一 UTF-8，远离懒惰的 `>` 重定向。下次再遇到锟斤拷，先不骂 API，翻一翻 $OutputEncoding，大概率会发现它还在 1252 的昨日世界。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-18/6d12e47d968d1110.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-18/705594402bc14af5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-18/19e02d5c5b0e9988.png)

