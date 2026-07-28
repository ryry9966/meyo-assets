---
title: Windows 上调用中文 JSON API？PowerShell 把中文打坏的真相与修复实录
feedId: 30839
source: 综合讨论
publishedAt: 2026-07-29
---

## 背景：Windows + 中文 API 的日常翻车

在 OpenClaw 的插件生态里，大量 Agent 和 MCP 工具需要调用 LLM 推理接口、内容检索接口或本地自动化 API。这些接口返回的 JSON 里常常包含中文摘要、标签或用户指令。很多 Windows 用户的自动化链条是：

> Python 构造请求 → PowerShell 脚本执行 curl 式调用 → 解析 JSON → 传给下一个 Agent

看似简单的链路，却会出现一个顽固问题：**接口明明返回了正常的中文，PowerShell 里看到的却是乱码，甚至因编码错乱导致 JSON 解析失败。** 有时只显示半个汉字，有时满屏 “鍝堝搱” 或空白，下游工具直接报 `invalid character` 或 `unexpected token`。

这个问题在不同 PowerShell 版本、不同终端里的表现还不一样，排查起来非常折磨。本文将梳理编码炸弹的引爆点，给出可靠的工程化方案，并附上可复现的配置步骤。

## 问题根源：三个编码层同时作祟

PowerShell 的字符串和管道处理涉及三层编码设置，任何一层不一致，中文就会坏掉。

1. **Web cmdlet 的响应解码层**  
   `Invoke-WebRequest` 和 `Invoke-RestMethod` 会尝试自动检测响应正文的编码。优先看 `Content-Type` 头里的 `charset`；如果没有，则回退到 ISO-8859-1。许多自建 API 返回 `content-type: application/json` 却不带 `charset=utf-8`，PowerShell 就按默认单字节编码去解释多字节的 UTF-8 中文，产生不可逆的乱码。

2. **控制台输出的转码层**  
   当你用 `Write-Host` 或直接输出对象时，PowerShell 会将内存中的 .NET 字符串（UTF-16）转码为控制台的代码页。Windows 控制台默认代码页是 936（GBK），而 PowerShell 的 `[Console]::OutputEncoding` 如果保持默认，可能与实际终端能力不匹配。这就是为什么同样的脚本在 VS Code 的终端里正常，在传统的 cmd 窗口或 PowerShell ISE 里却乱码。

3. **脚本文件与管道的编码层**  
   PowerShell 5.1 默认以 UTF-16 LE 保存脚本文件，执行时涉及 BOM 和编码转换。当脚本通过管道接收外部输出时，`$input` 的编码还受 `$OutputEncoding` 影响。如果 `$OutputEncoding` 不是 UTF-8，重定向或管道传输的中文就会丢失。

这三种编码各自独立，多数教程只告诉你改其中一个，结果换台机器就失效了。

## 做法/步骤：一套经测试的修补方案

以下步骤在 Windows 10/11 的 PowerShell 5.1 与 PowerShell 7.3 上验证通过，适合在 OpenClaw 工作流中固化。

### 1. 强制要求 UTF-8 响应

```powershell
# 使用 Invoke-RestMethod 时显式指定编码
$response = Invoke-RestMethod -Uri "https://api.example.com/items" `
   -ContentType "application/json; charset=utf-8" `
   -OutFile $null
# 如果仍然乱码，改用 Invoke-WebRequest 并手动解码
$bytes = (Invoke-WebRequest -Uri $url).Content | Out-String | 
         Select-Object -ExpandProperty InputObject
$decoded = [System.Text.Encoding]::UTF8.GetString([System.Text.Encoding]::Default.GetBytes($bytes))
# 更直接的方法：借助 HttpClient 强行 UTF-8
$http = New-Object System.Net.Http.HttpClient
$response = $http.GetAsync($url).Result.Content.ReadAsByteArrayAsync().Result
$json = [System.Text.Encoding]::UTF8.GetString($response)
```

当 `Invoke-RestMethod` 无法正确解码时，直接拿原始字节再手动转成 UTF-8 是最稳妥的。

### 2. 修复终端输出

在脚本开头显式声明输出编码：

```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$OutputEncoding = [System.Text.Encoding]::UTF8
```

两行都要写：第一行控制 `Write-Host` 和输出流的外显编码，第二行影响管道和重定向。需要注意，PowerShell ISE 不支持 `[Console]::OutputEncoding` 的修改，此时应切换到 VS Code 或新的 Windows Terminal。

### 3. 保存 JSON 结果到文件时用 UTF-8

```powershell
$json | Out-File -FilePath "./result.json" -Encoding utf8NoBOM  # PS7 使用 utf8NoBOM，PS5 用 utf8
```

PowerShell 5.1 的 `-Encoding utf8` 会带 BOM，某些下游 JSON 解析库会因此失败，可在 PS7 里避免。如果必须用 PS5，后续可加一步：

```powershell
(Get-Content "./result.json" -Encoding UTF8) | Set-Content "./result.json" -Encoding ASCII
```

不过 ASCII 会丢中文，更推荐直接用 `[System.IO.File]::WriteAllText("result.json", $json, [System.Text.UTF8Encoding]::new($false))` 写入无 BOM 的 UTF-8。

### 4. 统一脚本运行时的编码

如果你是调用外部 `.ps1` 文件，保存时选择 “UTF-8 无签名” 编码（VS Code 右下角可选择）。然后将以下语句放到脚本开头：

```powershell
$PSDefaultParameterValues['Out-File:Encoding'] = 'utf8'
$PSDefaultParameterValues['*:Encoding'] = 'utf8'
```

同时检查环境变量 `$env:LANG` 或 `$env:LC_ALL`（虽然 Windows 很少用），但 `chcp 65001` 可以在当前会话把控制台代码页切到 UTF-8：

```powershell
chcp 65001 > $null
```

## 踩坑点

- **PowerShell 5.1 的 `-ContentType` 实际不改变接收端解码方式**，它只是设置请求头。大多数乱码仍须手动处理响应。
- **Invoke-RestMethod 的 `-ResponseEncoding` 参数仅在 PS7 中可用**，PS5 里没有，只能走字节手动解码。
- **修改 `[Console]::InputEncoding`/`OutputEncoding` 会全局影响其他命令**，如果脚本频繁切换，可能引起其他模块异常。建议只在当前作用域修改，完成后恢复。
- **Docker 容器内跑 Windows 镜像时，默认代码页可能是 1252**，需要显式 `chcp 65001` 并设置 `$OutputEncoding`，否则日志中的中文同样损坏。
- **通过 `Start-Process` 调用新 PowerShell 实例时，不会继承 `$OutputEncoding`**，必须在新实例里再次设定。

## 可复用建议

1. **构建一个通用的 Safe-WebRequest 函数**，内嵌字节级 UTF-8 解码和文件保存逻辑，放到团队的模块里。
2. **在 OpenClaw 插件的 README 里明确要求**：运行环境使用 PowerShell 7+，并在脚本开头引入上面的编码修复代码块，避免 WinPS 5.1 的坑。
3. **使用 JSON Schema 校验响应**：在解码成字符串后立即用 `Test-Json` 验证，若失败就抛出明确错误，而不是让乱码数据污染下游 Agent 的上下文。

## 总结

PowerShell 把中文打坏，不是单个 Bug，而是 Windows 遗留代码页 + Web cmdlet 默认行为 + 终端转码三者耦合的结果。彻底修理需要从响应字节、内存字符串到输出终端全链路强制 UTF-8。这个设定一次到位后，后续的 Agent 调用、MCP 插件的往返消息都能稳定运行。把以上步骤写进你的部署脚本里，中文 JSON 不会再成为自动化链条上的断点。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-29/0823d09ffd24ea26.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-29/1e6cfb43fb784d23.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-29/5f5a104b26375b7e.png)

