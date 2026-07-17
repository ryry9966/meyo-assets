---
title: Windows 中文 JSON API 调用：为什么 PowerShell 会把中文打坏
feedId: 29457
source: 综合讨论
publishedAt: 2026-07-18
---

## 背景

在 Windows 上构建 Agent、MCP 工具或自动化脚本时，我们经常用 PowerShell 调用返回 JSON 的中文 API。典型场景是：`Invoke-RestMethod` 从 API 拿到包含书名、用户昵称或文章摘要的数据，接着写入文件、塞进提示词或通过 MCP 工具转发给下游。结果经常是——文件里中文变成问号或天书，下游 Agent 直接崩掉。

这个问题反复出现在 Windows Server 自动化、本地插件开发和 CI 流水线里，但它不是 PowerShell “坏了”，而是字符串从 HTTP 响应字节流到内存对象再到磁盘，中间经历了三次隐蔽的编码转换。

---

## 问题：中文 JSON 在 PowerShell 里烂掉的三个环节

### 1. 自动解析 JSON 时的“静默毁坏”
`Invoke-RestMethod` 会尝试自动解析 JSON，但它在 Windows PowerShell 5.1 下默认不会正确处理 UTF‑8 响应头。如果 API 没有明确返回 `Content-Type: application/json; charset=utf-8`，PowerShell 会按 **ISO-8859-1** 解码字节流。中文就此在内存对象里变成乱码，之后再怎么转码都已无法恢复。

### 2. `>` 或 `Out-File` 的编码陷阱
即使幸运地得到正确的字符串，当你用 `>` 输出到文件时，Windows PowerShell 5.1 的默认重定向编码是 **UTF-16 LE**，而很多后续工具（如 Python 脚本、Node.js MCP 客户端）期望 UTF-8。直接 `$json > result.txt` 会让整个文件带着 BOM 且宽字符无法直接兼容。

### 3. 控制台代码页与管道污染
有些脚本用 `[Console]::OutputEncoding` 来“修复”，但如果当前控制台代码页是 936（GBK），通过管道传递给外部命令时又会发生一次从 UTF-16 到 GBK 的隐式转换，导致二次损坏。

---

## 复现步骤（可靠触发乱码）

在一个 **中文版 Windows 10 / Server 2019** 上，使用 Windows PowerShell 5.1：

```powershell
# 一个真实返回中文 JSON 的测试 API（模拟常见情况）
$url = "https://httpbin.org/anything?text=你好世界"

# 天真写法
$data = Invoke-RestMethod -Uri $url
$data.args.text  # 输出：??? 或乱码

$data | ConvertTo-Json > result.json
# result.json 内容：缺少中文，或显示为 \u00e4\u00bd...
```

上述操作几乎必定复现乱码，根源是从字节到对象的第一次解码就错了。

---

## 正确的做法与工程化步骤

### 使用 `-ContentEncoding` 参数（适用于返回头明确为 UTF-8）
```powershell
$response = Invoke-WebRequest -Uri $url -ContentEncoding "UTF-8"
$obj = $response.Content | ConvertFrom-Json
$obj.args.text  # 此时正确
```

`Invoke-WebRequest` 配合 `-ContentEncoding` 能强制按 UTF-8 解码响应的原始字节，之后再送入 `ConvertFrom-Json` 解析成对象。

### 通用且更稳妥的修复：手动控制字节流解码
如果 API 不稳定，响应头里编码声明缺失或错误，最可靠的方式是自己拿到字节数组并解码：

```powershell
$respBytes = Invoke-WebRequest -Uri $url -UseBasicParsing | Select-Object -ExpandProperty Content
# 如果上面获取的是字符串，可能已损坏，改用下面：
$req = [System.Net.WebRequest]::Create($url)
$req.Method = "GET"
$resp = $req.GetResponse()
$stream = $resp.GetResponseStream()
$reader = New-Object System.IO.StreamReader($stream, [System.Text.Encoding]::UTF8)
$body = $reader.ReadToEnd()
$reader.Close()
$resp.Close()
$obj = $body | ConvertFrom-Json
```

### 输出文件时锁定 UTF‑8 without BOM
```powershell
$obj | ConvertTo-Json -Depth 10 | Out-File -FilePath result.json -Encoding utf8NoBOM
# 或使用 Set-Content
$jsonStr = $obj | ConvertTo-Json -Depth 10
[System.IO.File]::WriteAllText("result.json", $jsonStr, [System.Text.UTF8Encoding]::new($false))
```

> **注意**：`-Encoding utf8` 在 Windows PowerShell 中会带 BOM，很多工具不买账；`utf8NoBOM` 仅在 PowerShell Core 6+ 支持。更通用的方法是直接调用 .NET 的 `WriteAllText`。

---

## 踩坑点总结

1. **Invoke-RestMethod 自动解析不可信**：看不出数据何时被破坏，排查成本高。  
2. **控制台代码页的假象**：在 PowerShell 窗口内 `Write-Host` 显示正常，不代表内存数据正确——它可能只是显示时被控制台编码再次映射。  
3. **PowerShell 版本的坑**：Windows PowerShell 5.1 与 PowerShell 7+ 的默认编码行为完全不同。如果必须在旧版 Windows 上跑，要彻底避免依赖默认值。  
4. **管道给外部进程**：例如 `$json | python process.py`，此时数据被转换成 `$OutputEncoding`（默认为 ASCII 或系统代码页），中文同样会丢。务必将 `$OutputEncoding` 设为 `[System.Text.UTF8Encoding]::new()`。

---

## 可复用建议

在你的 MCP 工具或自动化插件启动脚本里（例如 `init.ps1`）显式加入这几行，可避免 90% 的编码问题：

```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$OutputEncoding = [System.Text.UTF8Encoding]::new()
$PSDefaultParameterValues['*:Encoding'] = 'utf8'
```

对任何对外 HTTP 调用，封装一个可信的函数 `Invoke-ApiSafe`，内部永远：
- 获取原始字节流
- 用 UTF‑8 解码为字符串
- 转成 JSON 对象
- 需要写入文件时，使用 `[System.IO.File]::WriteAllText(…, …, UTF8Encoding($false))`

---

## 总结

Windows 上的 PowerShell 并不是“中文杀手”，而是旧版默认编码与当代 UTF‑8 主流生态之间的断层，在 Agent 自动化中尤其需要显式管控。每一个隐式转换——HTTP 响应解码、管道编码、文件写入——都可能打断中文数据流，且往往静默。掌握字节→字符串→文件的完整链路，并采用强制 UTF‑8 的策略，才能让 Windows 下的中文 JSON 调用可靠地融入 OpenClaw、MCP 等现代化工具链。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-18/4493fe0f8775b2f6.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-18/b15b17b42ef036ab.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-18/47660f439bb88b1b.png)

