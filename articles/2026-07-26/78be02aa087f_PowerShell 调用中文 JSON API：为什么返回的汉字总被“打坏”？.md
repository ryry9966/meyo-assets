---
title: PowerShell 调用中文 JSON API：为什么返回的汉字总被“打坏”？
feedId: 30563
source: 综合讨论
publishedAt: 2026-07-26
---

在 Windows 上通过 PowerShell 调用中文 JSON API，几乎是每个做自动化（尤其是 OpenClaw/Agent/MCP/插件）的开发者早晚会踩到的坑。哪怕请求头带了 `charset=utf-8`，响应体里的中文在屏幕上看起来一切正常，一旦用脚本处理、写文件或传给下游，就会出现问号、乱码甚至意料外的转义序列。这篇文章从工程视角复盘问题根源，并给出一套可复用的修复方案。

## 背景：你在做什么

典型场景：你写了一个 PowerShell 脚本，通过 `Invoke-RestMethod` 或 `Invoke-WebRequest` 访问一个返回 JSON 的中文 API，例如：

```powershell
$resp = Invoke-RestMethod -Uri "https://api.example.com/zh/message" -Method Get
Write-Host $resp.content
```

控制台输出正常，但一旦你把 `$resp` 用 `Out-File` 存盘，或用 `ConvertTo-Json` 再传给下一个步骤，中文就变成了 `?`、`???`，或者一堆 `\uXXXX`。更糟的情况是，MCP 工具或 Agent 插件拿到乱码后直接执行错误逻辑，排查极其困难。

## 问题根源：PowerShell 的编码“双重人格”

Windows PowerShell（5.1）有两个关键编码设置，它们默认值与文本数据流之间的配合，正是乱码的来源。

1. **`[Console]::OutputEncoding`**  
   决定了控制台输出时的字节编码。中文 Windows 默认是 GBK（代码页 936），而不是 UTF-8。当你把 `$resp.content` 用 `Write-Host` 输出时，PowerShell 会尝试将 .NET 内部字符串（UTF-16）按照控制台编码转换成字节流，再交给终端渲染。由于 GBK 能正确显示中文，所以屏幕看到的是对的。

2. **`$OutputEncoding`**  
   决定了 PowerShell 通过管道（|）或重定向（>）向外部进程传递字符串时的编码。默认值是 ASCII 兼容的 `us-ascii`（Windows PowerShell 5.1）或 UTF-8（PowerShell 7+）。在 Windows PowerShell 中，管道传数据时，中文字符会被按照 ASCII 编码，非 ASCII 字符会变成 `?`。这就是脚本处理时乱码的根源。

3. **JSON 响应的实际编码**  
   大多数现代 API 返回的是 UTF-8 字节流，并在 HTTP 头中声明 `Content-Type: application/json; charset=utf-8`。`Invoke-RestMethod` 或底层的 `HttpClient` 能正确解码为 .NET 字符串。也就是说，当数据还停留在 `$resp` 这个对象里时，中文是完好无损的，只是后续输出路径编码出了问题。

## 典型踩坑表现

- **输出到文件**：`$resp.content | Out-File output.txt`  
  Windows PowerShell 默认使用 Unicode（UTF-16LE）写入文件，如果你用记事本打开可能没问题，但用 VSCode 或别的工具默认按 UTF-8 打开就会乱掉。更隐蔽的是，如果你显式指定 `-Encoding utf8`，但 PowerShell 5.1 的 `Out-File` 输出的 UTF-8 不带 BOM，于是一些工具无法正确识别，出现局部乱码。

- **将数据通过管道传给 Python/Node 脚本**：  
  `$resp.content | python helper.py` 在 Windows PowerShell 中 `$OutputEncoding` 为 ASCII，中文全部丢失。

- **在 MCP 插件里把返回值转成 JSON 字符串**：  
  使用 `ConvertTo-Json` 将对象序列化时，默认会转义非 ASCII 字符为 `\uXXXX`（除非加上 `-EscapeHandling EscapeNonAscii` 禁用），下游如果按字面解析，可能得到一堆 Unicode 转义序列，而不是直接可读的中文。

## 修复流程（可直接复制）

### 1. 统一使用 UTF-8 环境

在脚本开头强制设置输出编码：

```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$OutputEncoding = [System.Text.Encoding]::UTF8
```

这两行能解决大部分管道和外传问题。注意，`[Console]::OutputEncoding` 影响控制台显示，`$OutputEncoding` 影响管道。

### 2. 使用 `Invoke-RestMethod` 的安全参数

```powershell
$resp = Invoke-RestMethod -Uri $uri -Method Get -ContentType "application/json; charset=utf-8"
```

虽然不指定也不一定出错，但显式声明可以避免某些服务端不严谨的默认解析。

### 3. 将对象转为 JSON 时保持中文可读

```powershell
$jsonString = $resp | ConvertTo-Json -Depth 5 -EscapeHandling EscapeNonAscii
```

如果你希望 JSON 中直接保留中文（非转义），使用 `-EscapeHandling Default` 并配合 `-Compress` 控制格式。但传递给网络 API 时，转义形式也是合法 JSON，大多数客户端都能正确反序列化，所以只在需要人工阅读时调整。

### 4. 安全写入文件

永远不要依赖 `>` 重定向。使用：

```powershell
$resp.content | Out-File -FilePath "output.txt" -Encoding utf8
```

如果希望 BOM 也被生成（确保某些遗留工具识别），可以用 `[System.IO.File]::WriteAllText("output.txt", $resp.content, [System.Text.UTF8Encoding]::new($true))`。

### 5. 跨平台脚本建议

如果是 PowerShell 7+，它的默认编码行为（`$OutputEncoding` 为 UTF-8，`Out-File -Encoding` 也默认 UTF-8 且不带 BOM）更接近 Linux 习惯，但仍建议保持显式指定，避免在 Windows PowerShell 回退时踩坑。

## 高阶踩坑：MCP / Agent 插件上下文

当你的脚本作为 MCP 工具运行时，宿主进程（如 Claude Desktop、自定义 Agent）可能通过 stdin/stdout 与 PowerShell 通信。此时管道的编码至关重要。如果宿主期望 UTF-8，但 Windows PowerShell 默认 ASCII，就会导致中文参数丢失。最好的做法是：

- 用 PowerShell 7+ 作为执行引擎，并在脚本最顶部设置 `$OutputEncoding = [System.Text.Encoding]::UTF8`。
- 避免在 JSON 响应中混合多种编码路径（例如一段字符串先转字节再转回字符串），所有字符串操作一律在 .NET 的 UTF-16 语义下进行，只在最终输出边界做编码转换。

## 可复用建议

- **创建一个标准化启动块**：将所有编码修正放在一个 `profile.ps1` 或模块中，所有自动化脚本强制引入。
- **对所有外部 API 返回做编码健康检查**：如果怀疑数据被破坏，可以在内存中检查 `[System.Text.Encoding]::UTF8.GetBytes($str)` 是否包含替换字符 `0xEFBFBD`。
- **为团队固定 PowerShell 版本**：Windows 10/11 自带 PowerShell 5.1 和新版 PowerShell 7 并存，明确脚本需要的最低版本，并在脚本开头用 `#Requires -Version 7.0` 限制。
- **测试回显**：在初步调试时，打印 `$resp.content` 的长度、字节长度，以及特定汉字的 Unicode 码点，确保没有发生静默替换。

## 总结

PowerShell 把中文 JSON“打坏”，不是因为它处理不了中文，而是因为字符串在 .NET 内部完好，却在不同输出边界被不匹配的编码“翻译”丢失了信息。修复的核心在于将控制台输出编码、管道传递编码、文件写入编码全部对齐为 UTF-8。对于自动化实践者，与其每次排查乱码，不如将这些设置固化为基础设施的一部分，让脚本真正实现“一次编写，到处正确”。

---

