---
title: Windows 中文 JSON API 调用：为什么 PowerShell 会把中文打坏
feedId: 29573
source: 综合讨论
publishedAt: 2026-07-19
---

## 背景

在 OpenClaw、Agent 插件或 MCP（Model Context Protocol）这类自动化工具链中，通过 PowerShell 调用返回中文 JSON 的 HTTP API 是一个常见需求。典型场景包括：用本地脚本拉取企业内部服务的中文名称、调用大语言模型接口获得包含中文的响应、或者通过 REST 接口更新包含中文备注的配置项。

然而很多开发者在 Windows 下用 PowerShell 调用这类接口时，会得到一个既熟悉又令人沮丧的结果：中文字符变成了一串无法识别的乱码，有时是“锟斤拷”，有时是问号或者方块，甚至导致 JSON 解析失败。看起来像是“PowerShell 把中文打坏了”。这个问题并不罕见，但它背后涉及多个层次的编码配置，如果不理清根本原因，修复方案可能只是碰巧生效。

## 问题现象

假设有一个最简单的 REST API，返回如下 JSON：

```json
{
  "name": "中文测试",
  "message": "操作成功"
}
```

在 Windows 系统（大部分为 Windows 10/11 中文版）中使用 PowerShell 5.1 执行：

```powershell
$resp = Invoke-RestMethod -Uri "http://localhost/demo" -Method Get
$resp.name
```

控制台输出可能是 `???` 或者 `涓枃娴?`。如果将结果写入文件 `$resp | ConvertTo-Json | Out-File result.json`，再用其他编辑器打开，中文已经永久损坏。

更隐蔽的情况是 API 响应本身是 UTF-8，但后续拼接字符串再发送给另一个服务时，中文才会扭曲。这会直接影响 Agent 工作流中靠脚本串联的多个接口。

## 根因分析

PowerShell “打坏”中文并非单点故障，而是多条编码路径不一致叠加的结果。关键在于 Windows PowerShell（5.1 及更早版本）默认采用的是系统当前代码页（在中文 Windows 中通常是 GBK/代码页 936），而现代 API 几乎全部采用 UTF-8。当数据在这两种编码之间转换时，如果有一个环节使用了错误编码，就会产生乱码。

具体到调用链，有三个容易出问题的位置：

1. **Web Response 读取编码**  
   `Invoke-RestMethod` 和 `Invoke-WebRequest` 在接收 HTTP 响应时，会参考响应头中的 `Content-Type` 的 charset 字段。如果服务端正确返回了 `Content-Type: application/json; charset=utf-8`，PowerShell 会自动使用 UTF-8 解码，此时内存中的字符串是正确的。但很多内部服务、早期接口或者快速搭建的 mock 服务没有返回 charset，PowerShell 就会回退到 **ISO-8859-1** 解码，将 UTF-8 多字节流错误地解释为扩展 ASCII，导致中文字节被拆散。

2. **控制台输出编码**  
   即使 PowerShell 内存中的字符串已经是正确的 Unicode，在写入控制台或重定向到文件时，又会被转换成当前代码页。中文 Windows 控制台默认的输出编码是 GB2312/GBK（代码页 936），而 GBK 无法正确编码某些 Unicode 字符（尤其是使用了本地化扩展区的生僻字），导致输出时直接变成问号 `?`。此外，PowerShell 5.1 的重定向操作符 `>` 也会用 Unicode（UTF-16 LE）编码，但需要配合 `Out-File -Encoding` 参数明确指定，否则仍然沿用系统代码页。

3. **脚本文件自身编码**  
   如果 PowerShell 脚本文件 (.ps1) 本身保存为带 BOM 的 UTF-8，一切正常；如果保存为 ANSI 或 UTF-8 无 BOM，PowerShell 引擎可能错误解释中文字面量。更差的情况是，某些编辑器保存的“UTF-8”其实带有 BOM，而一些 API 在接收 JSON 字符串时，会把 BOM 当作正文的一部分，导致接口报错或中文开头出现一个不可见字符。

以上三个问题经常同时出现，造成排查时反复出现“明明改了这里，那里又出问题”的情况。

## 可行的解决方案

下面的方案基于 Windows PowerShell 5.1，但同样适用于 PowerShell 7，只是 PowerShell 7 默认行为更接近 UTF-8，问题不那么典型。这里以最容易出问题的 5.1 为例。

### 1. 强制 Web 响应使用 UTF-8 解码

避免依赖服务端 charset，主动在调用时指定编码。可以通过 `Invoke-WebRequest` 获取原始响应，再以 UTF-8 读取内容，最后手动解析 JSON。

```powershell
$response = Invoke-WebRequest -Uri "http://localhost/demo" -Method Get
$encoding = [System.Text.Encoding]::UTF8
$jsonString = $encoding.GetString($response.RawContentStream.ToArray())
$jsonString = $jsonString -replace '^.*?\r?\n\r?\n', ''  # 去掉 HTTP 头
$obj = $jsonString | ConvertFrom-Json
```

如果确认目标服务绝大多数情况返回 UTF-8，也可以在执行调用前全局修改 session 的默认字符串编码行为，但这属于侵入式修改，不推荐在插件或共享模块中使用。

另一种较温和的方式是使用 .NET 的 `HttpClient` 直接获取字节流，完全绕开 PowerShell 的自动编码猜测：

```powershell
$client = [System.Net.Http.HttpClient]::new()
$bytes = $client.GetByteArrayAsync("http://localhost/demo").Result
$obj = [System.Text.Encoding]::UTF8.GetString($bytes) | ConvertFrom-Json
```

### 2. 统一控制台和文件的输出编码

在脚本顶部或者模块初始化时，显式设置控制台的输出编码为 UTF-8：

```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
```

这会影响后续所有写入控制台的中文字符。但需要注意，如果终端窗口使用的字体不支持 UTF-8 字形（例如某些旧版 cmd 窗口），仍然会显示方块，此时需将终端切换到 Windows Terminal 或使用更现代的字体。

当需要把结果写入文件时，始终指定 `-Encoding UTF8`：

```powershell
$obj | ConvertTo-Json -Depth 10 | Out-File -FilePath result.json -Encoding UTF8
```

或者使用 `Set-Content -Encoding UTF8`。务必避免使用 `> result.json` 重定向，除非已经调整过 `$PSDefaultParameterValues` 中 Out-File 的默认编码。

### 3. 规范脚本文件的编码

- 使用 VS Code、Sublime 或 Notepad++ 等编辑器，将 `.ps1` 文件保存为 **UTF-8 with BOM**。  
- 如果团队强制要求不带 BOM（例如出于 git diff 或者跨平台脚本的考虑），则需要在脚本第一行加入一个特殊注释，并配合 PowerShell 7+ 使用。但在 Windows PowerShell 5.1 环境下，UTF-8 with BOM 是最安全的选择，可以避免中文字面量被错误解析。

### 踩坑记录

- **伪修复**：在 PowerShell 中执行 `chcp 65001` 只是修改了当前控制台的代码页，不影响 `Invoke-RestMethod` 的内部解码行为，对乱码根源没有任何帮助。
- **$OutputEncoding 陷阱**：`$OutputEncoding` 影响的是输出到外部命令（如 cmd.exe、python 等）时的编码，对控制台和文件输出没有直接效果，很多人将其与 `[Console]::OutputEncoding` 混淆。
- **StreamReader 默认编码**：如果使用 `StreamReader` 读取文件或响应流，必须显式传入 UTF-8 编码，否则它会采用系统默认代码页。
- **BOM 残害 JSON**：若脚本生成的中文 JSON 传递给一个严格的 JSON 解析器，且文件或字符串开头携带了 BOM（U+FEFF），会导致解析失败。生成 JSON 时尽量使用 `ConvertTo-Json` 配合管道输出到文件，而不是直接拼接字符串。

## 可复用的工程化建议

在编写需要调用中文 JSON API 的 PowerShell 模块或 MCP 工具时，可以封装一个通用函数：

```powershell
function Invoke-Utf8RestMethod {
    param(
        [string]$Uri,
        [string]$Method = 'Get',
        [hashtable]$Headers = @{}
    )
    $client = [System.Net.Http.HttpClient]::new()
    $response = $client.GetAsync($Uri).Result
    $bytes = $response.Content.ReadAsByteArrayAsync().Result
    $jsonText = [System.Text.Encoding]::UTF8.GetString($bytes)
    return $jsonText | ConvertFrom-Json
}
```

同时，在所有输出到文件的地方，集中定义一个 `Write-Utf8File` 函数，避免散落各处的 `Out-File` 忘记添加编码参数。对于控制台输出，可以在脚本最前部统一设置 `[Console]::OutputEncoding`，并在模块清单中建议使用者使用 Windows Terminal。

对于 OpenClaw / Agent 上下文，编码一致性还包括管道对接的其他进程（如 Python、Node.js 等都倾向于使用 UTF-8）。可以在 PowerShell 中设置环境变量：

```powershell
$env:PYTHONIOENCODING = 'utf-8'
$env:LANG = 'zh_CN.UTF-8'
```

降低整个自动化链路的编码不确定性。

## 总结

Windows PowerShell 处理中文 JSON API 时产生的乱码，本质是 UTF-8 数据在系统代码页、HTTP 响应解码、控制台输出三个环节中被误编码的结果。解决方案不能寄希望于单一魔法参数，而应当从强制 UTF-8 解码、显式输出编码、规范脚本文件编码三个方向同时加固。对于需要长期复用的自动化工具，建议将 UTF-8 处理逻辑封装为基础函数，从根本上杜绝“时而正常时而打坏”的随机性。在 Windows 生态里，拥抱 UTF-8 就是拥抱确定性。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-19/016baf073329cb64.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-19/ad9dfb4109c94dca.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-19/56d5f68d33afe1ee.png)

