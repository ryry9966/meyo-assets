---
title: 为什么 PowerShell 会把中文 JSON API 打坏？一次编码链的全景排查
feedId: 30468
source: 综合讨论
publishedAt: 2026-07-26
---

## 背景：从一段看似正常的脚本说起

在 OpenClaw-CN 的自动化实践中，不少社区成员会用 PowerShell 脚本调度 API 请求，比如让 Agent 查询内部带中文的知识库，或者用 MCP 插件调用某个返回中文摘要的服务。脚本逻辑很简单：

```powershell
$body = @{ prompt = "解释一下‘编码’的含义" } | ConvertTo-Json
Invoke-RestMethod -Uri "https://api.example.com/v1/chat" -Method Post -Body $body -ContentType "application/json"
```

结果要么服务端收到的“编码”变成了 `ç¼–ç `，要么返回的中文在终端里显示为一串方块或问号。更令人困惑的是，换成 `curl.exe` 直接在 PowerShell 里跑，中文同样面目全非。于是很多人怀疑是 Windows 对中文不友好，但其实问题出在 **PowerShell 的编码传送带**上。

这篇文章将沿着 API 调用的完整链路，把中文被打坏的原因讲清楚，并给出工程上可复用的解决方案。

## 问题定位：三条并行的编码通道

一次 API 调用至少涉及三处文本编码：

1. **PowerShell 进程内部**：脚本里声明的字符串以 .NET 的 `System.String` 形式存在，本质是 UTF-16 LE。
2. **跨进程边界**：当通过管道或命令行参数把字符串交给外部程序（比如 `curl.exe`）时，PowerShell 会依据 `$OutputEncoding` 和系统代码页进行一次转换。
3. **HTTP 通信层**：`Invoke-RestMethod` 或 `curl` 通过 HTTP 传输 JSON 正文，服务端按指定 charset 解析；返回的响应又可能携带另一种编码（如 GBK）。

乱码通常就发生在第②步和第③步的“翻译”阶段。

### 场景 A：直接用 `curl.exe` 传中文

很多老脚本习惯直接调 `curl`：

```powershell
curl.exe -X POST -H "Content-Type: application/json; charset=utf-8" -d '{"prompt":"中文测试"}' https://api.example.com/
```

在 PowerShell 中执行时，命令行参数 `中文测试` 会被按照 **系统当前活动代码页（简体中文环境下通常是 936，即 GBK）** 转换为字节流，然后由 `curl` 进程读取。`curl.exe` 默认会将这些字节当作 `ISO-8859-1` 或系统 ANSI 再解释为字符，这就会导致服务端收到乱码。如果你在脚本开头调用了 `chcp 65001` 把控制台切成 UTF-8，参数转换同样可能出错，因为 PowerShell 的 `$OutputEncoding` 默认还是 `US-ASCII`，二者不匹配时同样产生错误编码。

### 场景 B：`Invoke-RestMethod` 返回的中文乱码

`Invoke-RestMethod` 在处理响应时，会根据响应头中的 `Content-Type` 推断编码。但如果 API 返回的是一个未声明 charset 的 GB2312/GBK 文本，或返回头里写了 `charset=utf-8` 但实际却用了 GBK，`Invoke-RestMethod` 就会按照错误的编码将字节流解为字符串，从而导致显示乱码。

## 做法与步骤：三招根治中文编码病

### 1. 放弃「控制台编码混乱」的 curl，改用原生命令

这是最根本的解法。当你必须在 PowerShell 里调用 REST API 时，优先使用 `Invoke-RestMethod` 或 `Invoke-WebRequest`，并**显式指定字符集**。

**正确请求中文 JSON**：

```powershell
$payload = @{
    prompt = "解释一下‘编码’的含义"
}
$json = $payload | ConvertTo-Json -Compress

$response = Invoke-RestMethod -Uri "https://api.example.com/v1/chat" `
    -Method Post `
    -Body $json `
    -ContentType "application/json; charset=utf-8"
```

这里 `-Body` 接收的字符串会以 UTF-8 编码发出，无需担心系统代码页干扰。如果你必须传递包含中文字符的文件内容，可以用 `-InFile` 并确保文件本身是 UTF-8 with BOM（或使用 `-Body ([System.IO.File]::ReadAllBytes("file.json"))` 直接传字节数组）。

### 2. 处理非 UTF-8 响应：手动解码字节流

如果 API 顽固地返回 GBK 或 GB2312，可以这样兜底：

```powershell
$responseBytes = Invoke-WebRequest -Uri "https://api.example.com/legacy" -Method Get -ContentType "text/plain" | Select-Object -ExpandProperty Content
# Content 此时已是字符串，但可能是乱码，所以我们用 RawContentStream
$wr = [System.Net.WebRequest]::Create("https://api.example.com/legacy")
$wr.Method = "GET"
$resp = $wr.GetResponse()
$stream = $resp.GetResponseStream()
$reader = [System.IO.StreamReader]::new($stream, [System.Text.Encoding]::GetEncoding("gbk"))
$correctText = $reader.ReadToEnd()
$reader.Close(); $resp.Close()
```

或者更轻量地，如果你已经拿到字节数组（比如通过 `-OutFile` 或 `Invoke-WebRequest` 的 `RawContentStream`），直接用 `[System.Text.Encoding]::GetEncoding("gbk").GetString($bytes)` 转换。

### 3. 为整个脚本统一设定 UTF-8 环境

在脚本最顶部加入以下三行，可以防止大部分因控制台编码导致的隐式转换陷阱：

```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
[Console]::InputEncoding  = [System.Text.Encoding]::UTF8
$OutputEncoding = [System.Text.Encoding]::UTF8
```

注意：`$OutputEncoding` 影响 PowerShell 向外部命令发送数据的编码，`[Console]::OutputEncoding` 影响控制台显示的编码。两者都设为 UTF-8 后，再调用传统命令行工具的中文参数也会更安全。

## 踩坑点与避雷清单

1. **`ConvertTo-Json` 深度超过默认值**  
   PowerShell 5.1 的 `ConvertTo-Json` 默认深度为 2，嵌套对象可能被截断成字符串。务必加上 `-Depth 10` 或更大。

2. **BOM 幽灵**  
   一些 API 对 UTF-8 BOM 敏感。`ConvertTo-Json` 输出的字符串没有 BOM，但如果你用 `Out-File` 保存到文件，可能默认会带 BOM。建议用 `[System.IO.File]::WriteAllText("file.json", $json, [System.Text.UTF8Encoding]::new($false))` 保持无 BOM。

3. **用 `Start-Process` 调 curl 时参数引用**  
   如果实在必须用 `curl.exe`，请通过 `Start-Process -NoNewWindow -RedirectStandardInput` 之类的方式将 UTF-8 字节流直接送给标准输入，而不是通过命令行参数传递中文。

4. **Windows Terminal vs 传统 conhost**  
   终端不同会影响 `[Console]::OutputEncoding` 的默认值。Windows Terminal 默认 UTF-8，而传统控制台可能是 936。尽量在 Windows Terminal 中运行 PowerShell 7，并在脚本中显式设定编码，避免环境差异。

## 可复用建议

- **优先使用 PowerShell 7**：它对 UTF-8 的支持更开箱即用，`$OutputEncoding` 默认为 UTF-8，避免了很多 5.1 的历史包袱。
- **团队规范**：所有自动化脚本第一段统一为「环境初始化块」，包含编码设置、`$ErrorActionPreference = "Stop"` 等。
- **调试技巧**：遇到乱码时，可以用 `[System.Text.Encoding]::Default.EncodingName` 查看当前系统默认编码，用 `$OutputEncoding.EncodingName` 查看输出编码，帮助定位是哪一环出了问题。
- **单元测试**：在 CI/CD 流水线中增加一个包含中文固定 payload 的冒烟测试，确保不同机器上行为一致。

## 总结

Windows 上的中文编码问题不是玄学，而是 **PowerShell 进程内 UTF-16、跨进程 ANSI、网络层 UTF-8/GBK 三套编码相互转换时必然出现的摩擦**。只要坚持两条原则：

- **跨进程通信时，明确要求 UTF-8 并禁用隐式转换**；
- **接收数据时，永远不要信任响应头，做好按指定编码回退解码的准备**；

就能让 PowerShell 调用中文 JSON API 稳定得像在 Linux 上一样。这不仅是调通一个脚本的技巧，更是为整个 Agent/自动化流水线打下坚实的编码地基。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-26/482821988e37d842.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-26/2f74a6d08610b167.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-26/2247527bed3844fa.png)

