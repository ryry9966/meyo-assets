---
title: Windows 中文 JSON API 调用：为什么 PowerShell 会把中文打坏？
feedId: 30589
source: 综合讨论
publishedAt: 2026-07-27
---

## 背景

在 Windows 上写自动化，用 PowerShell 调用中文 API 几乎是个绕不开的场景——翻译、LLM 对话、OpenClaw 插件的 HTTP 工具链，都会在 PowerShell 里通过 `Invoke-RestMethod` 或 `Invoke-WebRequest` 抓回中文 JSON。可明明服务端返回的是 UTF-8 干净的 `{"text":"你好世界"}`，PowerShell 一接手，就变成了 `??????`、`ä½ å¥½ä¸-ç-`，甚至直接把后续的 JSON 解析搞崩。

这问题在 Agent 自动化流程中最致命：AI 模型拿到的上下文是乱码，指令中断，日志不可读，最后整个 MCP 调用链“无声失效”——没有报错，只有一堆无法理解的中文垃圾。这篇文章把编码踩坑的全流程、根因和可复用的修复方案整理出来，面向 Windows 环境下用 PowerShell 对接中文 API 的工程实践者。

## 问题：PowerShell 默认编码不是 UTF-8

核心矛盾在于三点：

1. **`Invoke-RestMethod` 的响应体解码默认跟随系统区域设置**。中文 Windows 的区域设置为“简体中文（中国）”，控制台代码页是 936（GBK），但很多 API 返回的是 UTF-8 编码的 JSON。当 PowerShell 用 GBK 去解释 UTF-8 字节流，多字节序列被拆散，出现乱码。
2. **`Out-File` 和 `>` （重定向运算符）默认输出 UTF-16LE**，而不是 UTF-8。这导致 Linux/macOS 工具读取日志、结果文件时又是一层乱码。
3. **`[Console]::OutputEncoding` 与 `$OutputEncoding` 的作用域及行为差异**，让许多“改编码”的补救方式看起来有效，换一个作用域就又失效了。

直接看起来像是“API 返回了乱码”，但根在 PowerShell 端。

## 做法/步骤

### 1. 重现与定位

用 `Invoke-RestMethod` 调一个公开的中文测试接口，例如：

```powershell
$resp = Invoke-RestMethod -Uri "https://httpbin.org/anything" -Method GET
$resp.data  # 若 data 内包含中文，大概率乱码
```

如果只想看原始字节而不触发自动解码，可以用 `Invoke-WebRequest` 加 `-ContentType` 显式标识，但更稳定的方法是拿 `Content` 的属性手动解码：

```powershell
$request = Invoke-WebRequest -Uri "https://httpbin.org/anything" -Method GET
$rawBytes = $request.RawContentStream.ToArray()  # 拿到完整响应
# 不过更简单是直接取 Content 再用正确的编码重新解码
$correctText = [System.Text.Encoding]::UTF8.GetString($request.Content.ReadAsByteArray())
```

注意 `RawContentStream` 包含 HTTP 头，处理起来麻烦，通常直接用 `Content` 属性——它返回的已经是解码后的字符串（乱码），所以需要回过头去拿原始的 `Content` 字节：通过 `$request.Content.ReadAsByteArray()` 即可获得字节数组，再用 UTF-8 重新构建。

### 2. 修复响应解码

**方案 A（全局设置）**：在脚本或 PowerShell 配置文件 `$PROFILE` 开头设置：

```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$OutputEncoding = [System.Text.Encoding]::UTF8
```

这两个变量意义不同：  
- `[Console]::OutputEncoding` 影响 PowerShell 控制台输出的编码；  
- `$OutputEncoding` 影响 `Invoke-RestMethod`/`Invoke-WebRequest` 对响应内容的解码方式。  
双管齐下，多数情况能直接解决。

对于 **Windows PowerShell 5.1**，还需要注意 ISE 和普通控制台的差异，最好在脚本最前面加一段检测：

```powershell
if ($PSVersionTable.PSVersion.Major -le 5) {
    $OutputEncoding = [System.Text.Encoding]::UTF8
}
```

**方案 B（更稳的工程化做法）**：不依赖默认解码，使用 `HttpClient` 或直接读取字节流。

在较复杂的 Agent 工具链里，我更倾向于抛弃 `Invoke-RestMethod` 的自动解析，改用 .NET `HttpClient`：

```powershell
$http = [System.Net.Http.HttpClient]::new()
$response = $http.GetStringAsync("https://api.example.com/translate").Result
```

此时 `GetStringAsync` 会遵循响应头中的 `charset`，若服务端写明 `Content-Type: application/json; charset=utf-8`，就能正确解码。同时还可以用 `GetByteArrayAsync` 拿到字节自己转。对于 OpenClaw MCP 风格的插件，这部分应封装成工具函数，统一处理编码。

### 3. 输出文件时强制 UTF-8

写入文件时避免使用 `>` 或默认 `Out-File`：

```powershell
$jsonResult | Out-File -FilePath "result.json" -Encoding utf8
```

或设置默认参数：

```powershell
$PSDefaultParameterValues['Out-File:Encoding'] = 'utf8'
$PSDefaultParameterValues['Set-Content:Encoding'] = 'utf8'
```

如果想用 `>>` 追加，要么写一个 wrapper 函数，要么干脆用 `Add-Content -Encoding utf8`。

## 踩坑点

- **`ConvertFrom-Json` 的深度限制**：嵌套很深的 JSON 会被截断，报错。需要传 `-Depth` 参数，建议直接给 `-Depth 100`。
- **`$response.Content | ConvertFrom-Json` 还是乱码**：因为 `$response.Content` 已经是乱码字符串，用 `ConvertFrom-Json` 解出来的属性值仍是错的。正确做法是先修复编码再解析。
- **Out-File 默认 UTF-16LE**：Unix 工具链读到一堆 BOM 和空字节，必须显式加 `-Encoding utf8`（无 BOM 用 `utf8NoBOM`，但需要 PowerShell 7+）。
- **`curl.exe` 别名捣乱**：在 Windows 10/11 上，PowerShell 默认 `curl` 实际上是 `Invoke-WebRequest` 的别名，而不是真正的 `curl.exe`。用 `curl` 跑中文 API 会掉进同样的编码陷阱，避免误用别名，直接调用 `curl.exe`。

## 可复用建议

1. **模板化脚本头**：每次建 `.ps1` 时自动插入以下片段：
   ```powershell
   # Ensure UTF-8 handling
   [Console]::OutputEncoding = [System.Text.Encoding]::UTF8
   $OutputEncoding = [System.Text.Encoding]::UTF8
   $PSDefaultParameterValues['*:Encoding'] = 'utf8'
   ```
2. **封装 HTTP 调用**：编写 `Invoke-ApiGetUtf8` 函数，内部用 `[System.Text.Encoding]::UTF8.GetString` 处理字节，返回正确字符串，然后再 `ConvertFrom-Json`。
3. **跨平台 agent 脚本**：如果是给 MCP/OpenClaw 插件用的脚本，在文档中注明 Windows 编码设置，并提供“启动前执行初始化”的选项，让宿主在运行前先 source 一个初始脚本。
4. **日志规范**：所有面向日志的输出都用 `Out-File -Append -Encoding utf8` 或 `Add-Content` 统一编码，避免 agent 回溯日志时再次踩坑。

## 总结

Windows 下 PowerShell 里中文 JSON 变乱码的根源是“编码不一致”：绝大多数现代 API 使用 UTF-8，而 PowerShell 在一些环节仍沿用旧的系统区域编码或 UTF-16LE 默认。自动化流程中，任何未经显式 UTF-8 声明的数据传递都可能让中文变成乱码，进而使上游 Agent 的行为完全不可预测。

**工程化的底线**：在调用任何中文 API 前，确保 `$OutputEncoding` 为 UTF-8，输出文件强制 `-Encoding utf8`，必要时放弃自动解析，直接通过字节流自行解码。这不仅是为了日志可读，更是为了保证 AI 驱动的决策链路不被无声扭曲。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-27/370ff1bb454f6100.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-27/7455b13431162872.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-27/125b8234c4c111e3.png)

