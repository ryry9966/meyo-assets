---
title: Windows 中文 JSON API 调用：为什么 PowerShell 会把中文打坏
feedId: 29997
source: 综合讨论
publishedAt: 2026-07-22
---

## 背景

在 OpenClaw 这类 Agent 自动化体系里，通过 PowerShell 调用中文 REST/JSON API 是高频操作——无论是获取天气、解析业务数据，还是桥接内部系统。然而，脚本在 Windows 上运行时经常出现中文变成 `锟斤拷`、`??` 或一串无法阅读的 Unicode 转义序列，直接导致后续解析失败或输出日志无法阅读。

这种现象并非代码逻辑错误，而是 **PowerShell 在 Windows 下的编码与管道输出机制** 引发的。本篇文章梳理问题根因、可复现的解决步骤，以及在实际插件/自动化脚本中的防御性写法。

---

## 问题复现

假设你在 Windows 10/11 上使用 Windows PowerShell 5.1，执行如下脚本调用一个返回中文 JSON 的接口：

```powershell
$response = Invoke-RestMethod -Uri "https://api.example.com/weather?city=上海"
$response | ConvertTo-Json | Out-File -FilePath result.json
```

打开 `result.json`，中文字段可能显示为乱码或 Unicode 转义（如 `\u4e0a\u6d77`），即便在控制台中直接打印 `$response` 时看起来正常。如果改用 `Invoke-WebRequest` 并访问 `.Content`，乱码依然存在。

更隐蔽的场景是：通过管道把结果传给外部工具（如 `jq`、`curl` 或用 `Start-Process` 传递参数），乱码在“最后一公里”出现，排错成本很高。

---

## 根因分析

核心矛盾在于 **PowerShell 对字符串的内部表示与输出编码不一致**。

1. **内部字符串编码**  
   PowerShell 的内存字符串是 .NET 的 `System.String`，本质是 UTF-16。从 API 收到的 JSON，`Invoke-RestMethod` 会正确解析成对象，此时中文无任何错误。

2. **输出/管道编码变量 `$OutputEncoding`**  
   Windows PowerShell 5.1 的 `$OutputEncoding` 默认值是 **ASCII (US-ASCII)**。当你把字符串通过管道传给外部程序，或者使用 `Out-File`、重定向操作符 `>` 时，PowerShell 会根据 `$OutputEncoding` 将内存中的 UTF-16 字符串重新编码成字节流。ASCII 无法表示中文，所有非 ASCII 字符会变成 `?` 或依据代码页进行错误转换。

3. **控制台显示编码 `[Console]::OutputEncoding`**  
   该变量只影响控制台主机的显示渲染，并不决定管道/文件输出的编码。即便你已经将其设为 UTF-8，`>$file` 仍然受 `$OutputEncoding` 控制。

4. **Invoke-WebRequest 的 `Content` 属性**  
   它的 `Content` 是一个字符串，.NET 在构造该字符串时默认使用响应的 `Content-Type` 头或 BOM 决定编码。如果服务器未明确指定 charset，可能使用 ISO-8859-1 或当前系统代码页（如 936），导致原始 UTF-8 字节被错误解码。

5. **PowerShell Core (v7+) 改善但未完全免疫**  
   PowerShell 7 的 `$OutputEncoding` 默认为 UTF-8（无 BOM），大幅降低了问题概率。但如果在管道中使用 `ForEach-Object` 等场景仍可能受控制台代码页干扰。

---

## 解决步骤与工程化做法

### 1. 优先使用 `Invoke-RestMethod`，输出时显式指定 UTF-8

```powershell
$result = Invoke-RestMethod -Uri "https://api.example.com/weather?city=上海"
$jsonStr = $result | ConvertTo-Json -Compress
[System.IO.File]::WriteAllText("result.json", $jsonStr, [System.Text.UTF8Encoding]::new($false))
```

这里 `UTF8Encoding($false)` 表示无 BOM，与大多数 API 的预期相符。`WriteAllText` 直接写入字节，绕过了 `Out-File` 的重编码链路。

### 2. 处理 `Invoke-WebRequest` 的原始字节

如果你需要检查响应头或必须用 `Invoke-WebRequest`，请手动解码：

```powershell
$resp = Invoke-WebRequest -Uri "https://api.example.com/weather?city=上海"
$bytes = $resp.RawContentStream.ToArray()
$jsonStr = [System.Text.Encoding]::UTF8.GetString($bytes)
$data = $jsonStr | ConvertFrom-Json
```

不推荐直接使用 `.Content`，除非你在请求中通过 `-ContentType "application/json; charset=utf-8"` 明确指定，并且服务器正确返回 charset。

### 3. 根治管道编码：设置 `$OutputEncoding`

在脚本顶部强制设定：

```powershell
$PSDefaultParameterValues['*:Encoding'] = 'utf8'
$OutputEncoding = [System.Text.UTF8Encoding]::new($false)
[Console]::OutputEncoding = [System.Text.UTF8Encoding]::new($false)
```

- 第一行让大部分 cmdlet 的输出文件编码默认为 UTF-8（PowerShell 5.1 有限支持，PowerShell 7 更可靠）。  
- 第二行解决管道到外部命令的编码。  
- 第三行保证控制台显示不乱码。

**注意**：在 Windows PowerShell 5.1 中，修改 `$OutputEncoding` 可能导致某些传统控制台程序（如 `findstr`）工作异常，建议仅在处理中文数据的脚本块中临时设置。

### 4. 使用 UTF-8 重定向的兼容写法

PowerShell 5.1 的 `> file` 重定向使用 `Unicode`（UTF-16 LE）编码，并非 UTF-8。因此要实现跨平台一致的输出，请用：

```powershell
$jsonStr | Out-File -FilePath result.json -Encoding utf8NoBOM
```

`-Encoding utf8NoBOM` 在 PowerShell 7 直接支持；PowerShell 5.1 需要 `-Encoding UTF8`（会带 BOM）。如果必须去 BOM，继续使用 `[System.IO.File]::WriteAllText`。

---

## 踩坑点汇总

1. **`ConvertTo-Json` 默认转义非 ASCII**  
   PowerShell 5.1 的 `ConvertTo-Json` 会将中文转义为 `\uXXXX`，看起来像乱码，但实际上是合法 JSON。如需原始字符，升级到 PowerShell 7 并使用 `ConvertTo-Json -EscapeHandling EscapeNonAscii` 或使用 `Newtonsoft.Json` 库。

2. **`Start-Process` 传递参数**  
   如果需要将中文作为命令行参数传给外部程序，参数编码受 `$OutputEncoding` 影响。确保预先设好 UTF-8，或改用管道传递标准输入。

3. **在 MCP/插件场景中通过标准输出传递 JSON**  
   如果你的 PowerShell 脚本作为 MCP 服务器通过 stdout 与宿主通信，宿主编解码默认通常是 UTF-8。若脚本使用了 `Write-Output` 或 `Write-Host`，在未设置 `$OutputEncoding` 的 Windows PowerShell 5.1 下会输出乱码。**强制改用 [Console]::OpenStandardOutput() 直接写入 UTF-8 字节流** 是最稳妥的方案。

4. **BOM 问题**  
   很多 API 或日志管道不接受 BOM 头，尤其在处理流式数据时。生成文件时明确写成无 BOM UTF-8。

---

## 可复用建议

- **封装 API 调用函数**：将编码设置、调用与解码逻辑封装在一个 `Invoke-ChineseApi` 模块中，避免每次重复配置。
- **环境检测**：在脚本开头检测 PowerShell 版本，如果是 5.1 则自动调整 `$OutputEncoding` 并显式写出无 BOM 文件。
- **CI/Agent 场景统一用字节流**：如果脚本输出 JSON 供下游 Agent 消费，直接输出字节流（例如 `[Console]::OpenStandardOutput().Write(byteArray, 0, byteArray.Length)`），确保编码完全可控。
- **测试与断言**：在自动化工程中加入编码测试，例如读取回文件断言中文字符长度，保证部署后不被环境变量意外覆盖。

---

## 总结

Windows 上 PowerShell 处理中文 JSON 的乱码，表面是编码问题，实质是 **输出管道 / 文件写入所依赖的 `$OutputEncoding` 与 API 实际编码（大多为 UTF-8）不匹配**。Windows PowerShell 5.1 的遗留默认值让这个问题根深蒂固，在构建 OpenClaw 类自动化或 MCP 插件时，忽视这一点会带来难以追踪的数据损坏。

核心对策：始终在脚本中显式设定输出编码为无 BOM UTF-8；优先用 `Invoke-RestMethod` 并直接写字节流；避免依赖 `>` 重定向和未指定编码的 `Out-File`。在团队内提供标准化的 API 工具函数，能有效避免这类隐性工程成本。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-22/e86e541b0fcb9fb9.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-22/5c9a14283c54702c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-22/222a0df67bb41853.png)

