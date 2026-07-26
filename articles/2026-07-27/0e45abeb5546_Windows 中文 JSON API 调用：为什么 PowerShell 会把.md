---
title: Windows 中文 JSON API 调用：为什么 PowerShell 会把中文打坏及根治方案
feedId: 30606
source: 综合讨论
publishedAt: 2026-07-27
---

## 背景

在 OpenClaw、Agent、MCP 这类自动化工作流里，PowerShell 经常被用来做最靠前的“胶水”：调用某个 REST API、抓回 JSON、再塞给下游工具。一旦返回的数据包含中文字段，就很容易出现一个看似玄学的问题——**终端里明明字符正常，可一旦重定向到文件、传递给 MCP 或写入管道，中文就变成乱码甚至全是问号**。

这个问题并不是 PowerShell 的 bug，而是 Windows 下 PowerShell 历史遗留的编码策略在自动化场景中被反复踩中。以下从一个真实调试过程出发，拆解原因并给出可工程化复用的修复方式。

## 问题：中文 JSON 如何在管道中“打坏”

假设有一个天气 API，返回类似 `{"city":"北京","condition":"晴"}` 的 JSON。在 Windows PowerShell 5.1 中执行：

```powershell
$resp = Invoke-RestMethod -Uri "http://httpbin.org/anything" -Body '{"city":"北京"}' -Method Post
$resp.json.city  # 控制台正常显示“北京”
$resp | ConvertTo-Json -Compress | Out-File result.json
# 再用记事本打开 result.json，city 变成了 “??”
```

即使使用 `Invoke-WebRequest` 再取 `.Content`，直接用 `Set-Content` 或 `>` 重定向，同样会损坏中文；并且这个损坏不是终端的“显示问题”——事实上写入文件的字节已经错误，后续任何 JSON 解析都会失败。

根因有两个层面：  
1. **输出到文件时的默认编码**。Windows PowerShell 5.1 的 `>` / `Out-File` 默认使用 **UTF-16LE**（带 BOM）；而 `Set-Content` 默认使用系统 ANSI 代码页（如 GBK）。如果你的 API 返回的是 UTF-8 字符流，这些重编码会把多字节字符割裂。  
2. **控制台输出编码与管道编码解耦**。`[Console]::OutputEncoding` 只影响控制台渲染，但当数据通过管道传递或写入文件时，真正起作用的编码是 `$OutputEncoding`（管道编码）以及 cmdlet 自身的默认值。很多教程只教人改控制台编码，但这对管道输出无效。

## 复现场景

在任何一台默认编码的 Windows 10/11 上，使用 PowerShell 5.1：

```powershell
$json = '{"msg":"你好"}'
$obj = $json | ConvertFrom-Json
$obj.msg             # 控制台正常显示“你好”
$obj | ConvertTo-Json | Out-File a.json
# a.json 内容：{"msg":"??"}
```

如果使用 `>` 重定向：

```powershell
$obj | ConvertTo-Json > b.json
# b.json 是 UTF-16LE 带 BOM，部分工具可能仍然能解析，但编辑器中看到乱码
```

当这个 `b.json` 被 MCP 的 `read_file`、Python 的 `json.load()`、或 Node.js 的 `fs.readFileSync` 直接读取时，就会解析失败或出现中文乱码。

## 根治做法 / 步骤

### 1. 统一使用 UTF-8 编码（无 BOM）

对于 **PowerShell 7+**（推荐优先使用），直接指定 `-Encoding utf8NoBOM`：

```powershell
$obj | ConvertTo-Json -Compress | Out-File -Encoding utf8NoBOM result.json
# 或
$obj | ConvertTo-Json -Compress | Set-Content -Encoding utf8NoBOM result.json
```

对于 **Windows PowerShell 5.1**，`-Encoding utf8` 会强制带 BOM，部分 JSON 解析器视为非法。消除 BOM 需用 .NET 方法：

```powershell
$jsonStr = $obj | ConvertTo-Json -Compress
[System.IO.File]::WriteAllText(
    "result.json",
    $jsonStr,
    [System.Text.UTF8Encoding]::new($false)
)
```

### 2. 修正管道与输出编码

在脚本最顶部增加：

```powershell
$OutputEncoding = [System.Text.UTF8Encoding]::new($false)
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
```

`$OutputEncoding` 决定把数据发送给外部进程或管道时的编码；`[Console]::OutputEncoding` 确保控制台不产生二次编码混乱。这样即使某些 cmdlet 调用默认行为，也可减少意外。

### 3. 使用 `Invoke-RestMethod` 时直接操作对象

尽量避免把响应存为字符串再重编码。PowerShell 会自动将 JSON 响应解析为 PSObject，直接使用对象属性传递数据，只在最终落盘时才转化为字符串并强制 UTF-8 NoBOM。如此路径最短，受编码影响最小。

## 踩坑点汇总

- **`Out-File -Encoding utf8` 在 PS 5.1 总是带 BOM**，这一点在官方文档里不易察觉，实际调试才会发现。  
- **`Set-Content` 和 `Add-Content` 默认编码是 ANSI**，而不是 UTF-8，即使系统区域设置为中文也会导致掉字。  
- **有些文章建议修改 `$PSDefaultParameterValues` 全局设置 `Out-File:Encoding`，但仅影响当前会话，且对内部 cmdlet 的字符处理无效**，在实际 Agent 调用脚本时容易遗忘，导致回退到默认值。  
- **IDE（VS Code / ISE）内嵌终端默认编码可能与外部 PowerShell 不同**，导致“在我机器上是好的”错觉。建议单独在外部 PowerShell 窗口中验证。  
- **MCP 的 `execute_command` 工具如果捕获 PowerShell 的 stdout，通常期望 UTF-8 NoBOM**。此时哪怕丢失一个字节，MCP 返回给 Agent 的文本就会损坏，触发 “JSON 解析失败” 这种模糊错误，掩盖了根因。

## 可复用建议

1. **所有自动化的 PowerShell 脚本模板开头固定写**：  
```powershell
# encoding rescue block
$OutputEncoding = [System.Text.UTF8Encoding]::new($false)
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$PSDefaultParameterValues['Out-File:Encoding'] = 'utf8'
```
并使用 `.NET 写入无 BOM` 作为文件输出通用方法。

2. **优先使用 PowerShell Core (7+) 的 `utf8NoBOM`**，可减少 BOM 引发的下游异常。  
3. **在 Agent 工作流中，避免让 PowerShell 直接输出到文件再读回**；尽量让脚本返回结构化对象，由宿主语言（如 Python）完成序列化落盘。如果必须落盘，可调用一个极小的 Python 脚本来保证编码一致。  
4. **测试时，用 `xxd` 或 `Format-Hex` 检查实际写入的字节**：  
```powershell
Format-Hex result.json -Count 64
```
若看到字节序列并非有效 UTF-8，即可快速定位。

## 总结

PowerShell 在 Windows 上的中文 JSON 乱码，常被误以为是 API 本身的问题，实际是连续输出路径上的编码策略不一致。在自动化 / OpenClaw / MCP 场景中，任何一个中间工具如果生成了带有错误编码或多余 BOM 的文件，就会在 Agent 不可见的地方打断数据流。通过强制输出无 BOM UTF-8、修正 `$OutputEncoding`、并尽可能直接操作对象，可以彻底根治此类“隐形”故障，让工具链稳定闭环。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-27/7943e17b934265a5.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-27/e7325cffdce242ab.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-27/6cef495d4cd108eb.png)

