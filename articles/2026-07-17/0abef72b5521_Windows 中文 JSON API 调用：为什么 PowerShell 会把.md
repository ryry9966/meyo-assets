---
title: Windows 中文 JSON API 调用：为什么 PowerShell 会把中文打坏，以及如何让编码不再“吞字”
feedId: 29430
source: 综合讨论
publishedAt: 2026-07-17
---

## 背景：Agent/MCP 插件里的“隐形炸弹”

在 OpenClaw 这类 Agent 工作流里，经常需要通过本地脚本或工具调用外部 API，再把结果喂给大模型。Windows 环境下，PowerShell 因为集成度高、无需额外安装，常常是写 glue script 的首选。但一旦 API 返回中文，你会发现终端里看到的正常内容，经过脚本处理后却变成了乱码、问号甚至直接截断。更糟的是，这种错误不一定每次都发生，尤其在你用 `Invoke-RestMethod` 或 `Invoke-WebRequest` 直接输出到文件时，乱码可能会被悄悄写入，最后导致 Agent 解析 JSON 失败，或者生成错误的上下文。

这个问题本质上是 Windows 下 PowerShell 的字符编码协商机制和 .NET 字符串内部表示的冲突。帖子不会重复那些“改成 UTF-8”的泛泛之谈，而是要给出一个可复现、能用在自动化流水线里的维修方案。

## 问题复现：从“看起来正常”到“实际坏了”

典型场景：你用 PowerShell 调用一个中文 API，例如：

```powershell
$response = Invoke-RestMethod -Uri "https://api.example.com/zh-cn/data" -Method Get
$response | ConvertTo-Json -Depth 3 | Out-File -FilePath "output.json"
```

用记事本打开 `output.json`，中文显示正常，但如果你用 `type output.json` 重定向给另一个程序，或者用 Python 读取这个文件，就会发现文件头缺少 BOM，编码其实是 `UTF-16 LE` 或者系统代码页对应的 ANSI（如 GBK）。更隐蔽的是，PowerShell 7 虽然默认输出 UTF-8 without BOM，但某些 Windows 旧模块或 `cmdlet` 会退回到系统 OEM 代码页，导致流水线里读取文件时乱码。

另一个经典坑：在 PowerShell 5.1（Windows 内置版本）中，重定向操作符 `>` 实际上是 `Out-File` 的别名，默认使用 **Unicode (UTF-16LE)** 编码。因此即使你看见终端输出正常，`python script.py > output.txt` 也可能产生一个 UTF-16LE 文件，而很多 Linux 原生的解析器并不认识它。

## 为什么 PowerShell 会打坏中文？编码协商的三个层次

要真正解决问题，得理解三个层次：

1. **控制台宿主编码**：`[Console]::OutputEncoding` 决定了 PowerShell 向管道或文件写入文本时使用的编码。Windows 控制台通常默认是系统 OEM 代码页（如 936 对应 GBK）。
2. **字符串内部编码**：PowerShell 内部所有字符串都是 .NET 的 `System.String`，即 UTF-16。理论上只要正确序列化就不会丢失信息，但从内部到外部写入时需要一次编码转换。
3. **cmdlet 的默认编码参数**：`Out-File`、`Set-Content`、`Export-Csv` 等 cmdlet 在 Windows PowerShell 5.1 中默认使用 **Unicode (UTF-16LE)** 或 **ASCII**，而 PowerShell 7 改为 **UTF-8 without BOM**。这一不一致导致跨版本脚本经常撞墙。

一个容易忽略的点是：**`Invoke-RestMethod` 返回的 `Content` 编码取决于服务器响应的 `charset`**。如果服务器返回的 JSON 没有声明 `charset=utf-8`，PowerShell 可能会根据 BOM 自动猜测，或用 `ISO-8859-1` 解码，把多字节的中文拆成乱码。此时即使后续保存为 UTF-8，也已经不可逆地损坏了。

## 做法：端到端固定编码的铁三角方案

以下是经过多个 OpenClaw 插件项目验证的可靠做法，适用于 Agent 的 tool script。

### 1. 显式控制 `Invoke-RestMethod` 的响应编码

放弃依赖自动检测，使用 `Invoke-WebRequest` 拿到原始字节，然后按 UTF-8 解码：

```powershell
$response = Invoke-WebRequest -Uri "https://api.example.com/zh-cn/data" -Method Get
$rawBytes = $response.RawContentStream.ToArray()
$jsonString = [System.Text.Encoding]::UTF8.GetString($rawBytes)
$data = $jsonString | ConvertFrom-Json
```

如果确定服务器返回 UTF-8，也可以设置 `$response.Content` 但强制指定：

```powershell
$response = Invoke-WebRequest -Uri $url
$encoding = [System.Text.Encoding]::UTF8
$reader = New-Object System.IO.StreamReader($response.RawContentStream, $encoding)
$jsonString = $reader.ReadToEnd()
```

这避免了因缺失 charset 导致的解码错误。

### 2. 统一所有写入操作为 UTF-8 with BOM（兼顾 Windows 兼容）

对于 Agent 需要的中间文件，使用 BOM 可以避免记事本和旧工具误判编码：

```powershell
$data | ConvertTo-Json -Depth 10 | Out-File -FilePath "output.json" -Encoding utf8
```

注意：PowerShell 5.1 中 `-Encoding utf8` 生成的正是 **UTF-8 with BOM**。这样既能让记事本正确识别，也能被大多数跨平台工具接受。如果你的下游是纯 Linux 环境，且不需要在 Windows 上手动查看，可以用 `-Encoding utf8NoBOM`（仅 PowerShell 7 支持）。

### 3. 在脚本开头锁定进程级编码变量

一条容易被忽略但极其有效的指令：

```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$PSDefaultParameterValues['Out-File:Encoding'] = 'utf8'
$PSDefaultParameterValues['*:Encoding'] = 'utf8'
```

第一条让控制台输出编码与管道行为一致；后两条为所有支持 `-Encoding` 的 cmdlet 设定默认值，避免每次都要加参数。这对于经常在不同 Windows 主机上运行的 Agent 脚本尤其重要。

## 踩坑点：不是所有 UTF-8 都是一样的

- **Invoke-RestMethod 的自动解析**：很多人以为用 `-ContentType "application/json; charset=utf-8"` 发送请求就够了，但这只影响请求头，不会控制对响应的解码。
- **Windows 下的 `curl.exe` 别名**：PowerShell 5.1 将 `curl` 别名为了 `Invoke-WebRequest`，而真正的 `curl.exe` 行为不受 PowerShell 编码控制。如果你混用两者，编码环境可能不一致。建议在脚本中明确调用 `curl.exe` 或 `Invoke-WebRequest`。
- **管道间的隐式编码转换**：当使用 `|` 把对象传给外部命令时，PowerShell 会将对象转换为字符串，转换所用的编码就是 `[Console]::OutputEncoding`。如果不设成 UTF-8，中文经过管道就变成 `?`。

## 可复用建议

把以下代码块保存为 `lock-utf8.ps1`，在每个需要处理中文的 tool script 开头 dot-source 引入：

```powershell
# lock-utf8.ps1
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$PSDefaultParameterValues['Out-File:Encoding'] = 'utf8'
$PSDefaultParameterValues['Set-Content:Encoding'] = 'utf8'
$PSDefaultParameterValues['Export-Csv:Encoding'] = 'utf8'

# 对于 Invoke-WebRequest 封装一个安全函数
function Invoke-WebRequestUtf8 {
    param([string]$Uri, [string]$Method = 'Get')
    $resp = Invoke-WebRequest -Uri $Uri -Method $Method
    $reader = New-Object System.IO.StreamReader($resp.RawContentStream, [System.Text.Encoding]::UTF8)
    return $reader.ReadToEnd() | ConvertFrom-Json
}
```

在 OpenClaw/Agent 的 action 配置中，调用脚本时确保 `powershell.exe -File your_script.ps1`，并在脚本开头 `. .\lock-utf8.ps1`，就可以让中文稳定通透。

## 总结

PowerShell 把中文打坏，不是因为“微软不重视国际化”，而是因为 Windows 生态里编码兼容负担太重，导致默认值倾向于历史代码页而非现代化的 UTF-8。在 Agent 工具链中，我们既是调用方也是数据搬运工，必须显式控制写入、读取、管道三个环节的编码，并尽可能锁定 `Console]::OutputEncoding` 和默认 cmdlet 编码参数，才能让中文 JSON API 调用真正稳定。记住：**凡是流入大模型上下文的数据，编码小错误足以让推理偏离千里**。

---

