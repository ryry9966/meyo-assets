---
title: Windows 中文 JSON API 调用的隐形杀手：为什么 PowerShell 会把你的 UTF-8 打坏
feedId: 30483
source: 综合讨论
publishedAt: 2026-07-26
---

在 OpenClaw、Agent 或自定义 MCP 插件的实践中，经常需要从 PowerShell 脚本调用返回中文 JSON 的 API，再将结果转发给下游工具。这类流水线看似简单：一条 `Invoke-RestMethod` 获取数据，一段 `ConvertTo-Json` 结构化，最后 `Out-File` 落盘或管道传递给下一个进程。然而，Windows 环境下的开发者几乎都会撞上同一个问题——中文在某个环节变成了乱码、问号或 `\uXXXX` 转义序列，轻则输出不可读，重则导致下游解析失败。更迷惑的是，控制台直接打印时汉字一切正常，只有写入文件或重定向时才“打坏”。本文还原问题的根因，给出可复现的修复路径，并提炼出在自动化脚本中彻底避免此类编码故障的工程实践。

## 背景：PowerShell 的编码多态性

Windows 上通常同时存在两个 PowerShell：系统自带的 Windows PowerShell 5.1（以下简称 PS5）与通过 MSI 安装的 PowerShell 7+（即 PowerShell Core，以下简称 PS7）。二者在字符编码的默认行为上有本质差异：

- **PS5 的默认输出编码**：基于旧式 Windows 控制台（代码页），重定向操作符 `>` 和 `Out-File` 在没有指定 `-Encoding` 时，使用的是 **Unicode（UTF-16LE）** 编码，而非常见的 UTF-8。此外，当与外部本机程序通过管道交换数据时，`$OutputEncoding` 变量默认为 ASCII（代码页 20127），这会导致非 ASCII 字符被静默破坏。

- **PS7 的现代化改进**：重定向和文件输出默认采用 **UTF-8 无 BOM**，`$OutputEncoding` 也默认为 UTF-8。然而，多数企业环境与自动化脚本仍被迫跑在 PS5 上，这就埋下了编码地雷。

当 API 返回的是 UTF-8 编码的 JSON（最常见场景），并且 JSON 内包含中文时，PS5 的默认行为就会在“输出到文件”或“通过管道交给另一个进程”这两个节点触发编码断裂。

## 问题复现：从正常到乱码的完整路径

假设一个天气 API 返回的 JSON 为：

```json
{"city":"北京","temperature":"26°C"}
```

我们在 PS5 脚本中如此调用：

```powershell
$response = Invoke-RestMethod -Uri $apiUrl
$response.city  # 控制台输出：北京 （正常）
$response | ConvertTo-Json | Out-File -FilePath result.json
```

打开 `result.json`，你可能会看到：

```json
{
    "city": "??",
    "temperature": "26�C"
}
```

或者如果是用 `>` 重定向：

```powershell
$response | ConvertTo-Json > result.json
```

文件编码变成 UTF-16LE，某些文本编辑器无法正确解析，显示为小方块或乱码。

**故障链分析：**

1. `Invoke-RestMethod` 正确解析了服务器返回的 UTF-8 流，将 JSON 反序列化为 .NET 对象，字符串在内存中以 .NET 内部 UTF-16 形式存储，无问题。
2. `ConvertTo-Json` 将对象序列化为 JSON 字符串，默认行为是在内存中生成 .NET 字符串，仍保持汉字，但**默认将非 ASCII 字符转义为 `\uXXXX` 序列**（PS5 的行为）。若你想看到汉字，需加 `-Compress` 或 PS7 的 `-Encode 0`，但最关键的问题还在后头。
3. `Out-File -FilePath result.json` 在 PS5 下**默认使用 Unicode（UTF-16LE）编码写入**。如果下游工具期望 UTF-8，文件头部会多出 BOM，且字节序列完全不同，乱码出现。
4. 如果你尝试修正 `Out-File` 的编码，写成 `Out-File -Encoding UTF8`，PS5 会写入**带 BOM 的 UTF-8**，而很多 Linux 端工具或 JSON 解析器对 UTF-8 BOM 过敏，可能解析失败或留下不可见字符。
5. 管道输出到外部程序（如 `curl.exe`、Python 脚本）时，PS5 根据 `$OutputEncoding` 将字符串转为字节流，默认 ASCII，汉字变成 `?`。

## 根治方案与可复用工程实践

**原则：在脚本开始处强制统一编码环境，不要依赖任何默认值。**

### 1. 固化输出编码为无 BOM 的 UTF-8

在 PS5 中，推荐使用 `Set-Content` 替代 `Out-File`，并显式指定 UTF-8 无 BOM：

```powershell
$json = $response | ConvertTo-Json -Depth 10
[System.IO.File]::WriteAllText("$PWD/result.json", $json, [System.Text.UTF8Encoding]::new($false))
```

其中 `[System.Text.UTF8Encoding]::new($false)` 构造一个无 BOM 的 UTF-8 编码器。这样写出的文件是纯正的 UTF-8，不会给下游造成 BOM 困惑。

若必须使用 `Out-File`（比如为了追加内容），请结合 `$PSDefaultParameterValues` 预设参数：

```powershell
$PSDefaultParameterValues['Out-File:Encoding'] = 'utf8'
$PSDefaultParameterValues['*:Encoding'] = 'utf8'
```

但在 PS5 中 `Out-File -Encoding utf8` 依然带 BOM，可以用 `Out-File -Encoding ascii` 避免 BOM 却又无法包含中文，所以建议直接使用 .NET 文件写入方法。

### 2. 修正管道编码

在脚本顶部设置：

```powershell
$OutputEncoding = [System.Text.Encoding]::UTF8
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
```

- `$OutputEncoding`：影响 PowerShell 将字符串发送给外部程序时的编码。
- `[Console]::OutputEncoding`：影响控制台显示和某些命令行工具的交互。

这样当脚本通过管道将 JSON 文本传递给 `curl` 或 `python` 时，中文能完好无损地以 UTF-8 字节流传递。

### 3. 关闭 ConvertTo-Json 的 Unicode 转义

在 PS5 中，`ConvertTo-Json` 默认会将中文字符转义为 `\u5317\u4eac`，虽然 JSON 标准允许，但人类难以阅读，也增加体积。如果你希望保留原始汉字，可借助辅助函数：

```powershell
function ConvertTo-JsonUnescaped {
    param($InputObject, $Depth = 10)
    $json = $InputObject | ConvertTo-Json -Depth $Depth
    # 解码 \uXXXX 转义序列
    [System.Text.RegularExpressions.Regex]::Unescape($json)
}
$json = ConvertTo-JsonUnescaped -InputObject $response
```

在 PS7 中直接用 `ConvertTo-Json -Encode 0` 即可。

### 4. 统一脚本运行环境检测

在脚本开头增加环境检测，当发现是 PS5 且未正确设置编码时，发出警告或自动修正：

```powershell
if ($PSVersionTable.PSVersion.Major -lt 6) {
    Write-Warning "Running on Windows PowerShell 5.1, applying UTF-8 fixes."
    $OutputEncoding = [System.Text.Encoding]::UTF8
    [Console]::OutputEncoding = [System.Text.Encoding]::UTF8
}
```

## 踩坑点总结

- **BOM 地雷**：PS5 的 `Out-File -Encoding UTF8` 会生成带 BOM 文件，某些 JSON 解析器（如 `jq`）直接报错。务必用 `[System.Text.UTF8Encoding]::new($false)` 避开。
- **重定向陷阱**：`>` 在 PS5 中调用 `Out-File` 并采用 Unicode 编码，不是 UTF-8。哪怕你已设置 `$OutputEncoding`，重定向操作仍忽略该变量。避免使用 `>` 输出中文 JSON 到文件。
- **Invoke-WebRequest 的 Content 区别**：如果手动用 `Invoke-WebRequest` 获取内容，再通过 `.Content` 取字符串，PS5 可能已根据响应头错误解码。优先使用 `Invoke-RestMethod` 直接得到对象。
- **跨平台协作**：如果脚本需在 Linux/macOS 的 PS7 上运行，上述修复不会产生副作用，因为 `$false` 的 BOM 设置在这些平台本身就是默认行为。这使得修复具有可移植性。

## 总结

PowerShell 在 Windows 上的编码行为源于对旧式控制台和 Windows API 的兼容，这种“多态”在自动化流程中很容易将中文 JSON 打坏。问题的本质是：内存中的字符串永远是正确的，但一旦触及文件写入或跨进程管道，PS5 的默认编码选择就不再适应当下 UTF-8 标准化的世界。

工程化的解决方案不是祈祷用户升级到 PS7，而是在每一个输出边界上显式声明 UTF-8 无 BOM，并固化 `$OutputEncoding`。当你的 OpenClaw 插件、Agent 工作流或 MCP 连接器需要稳定运行在生产环境时，这层保护就是防止数据腐坏的最后一道防线。

记住：**任何不显式声明编码的 PowerShell 脚本，都是留给未来的地雷——中文爆炸只是时间问题。**

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-26/ae9f7dc8680f3d14.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-26/4f4acc257ada039b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-26/9e949d1f63f51821.png)

