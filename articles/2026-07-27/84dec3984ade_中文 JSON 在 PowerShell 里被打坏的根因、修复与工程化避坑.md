---
title: 中文 JSON 在 PowerShell 里被打坏的根因、修复与工程化避坑
feedId: 30639
source: 综合讨论
publishedAt: 2026-07-27
---

# 中文 JSON 在 PowerShell 里被打坏的根因、修复与工程化避坑

## 背景：Windows + PowerShell + 中文 API = 暗坑标配

在 OpenClaw、Agent、MCP 或自动化插件场景中，我们经常需要让脚本去调一个返回中文 JSON 的 HTTP API，然后用 PowerShell 把结果存成文件、传给下一个工具或直接解析。很多人在 Windows 上写完脚本测试时发现：同一个 API，在 Postman 里看到的中文完全正常，到了 PowerShell 脚本里就变成了 `????`、`錕斤拷` 或者整段丢失。

这不是“偶发的编码问题”，而是 Windows 平台上 PowerShell 5.1 的默认行为与 UTF-8 生态之间的一次系统性冲突。如果你用的是 GitHub Actions 的 Windows runner、自建 Windows Agent 或者一台装了中文语言包的虚拟机，这个坑几乎一定会出现。

## 问题：谁在打坏中文？

先看一个最简的复现场景。用 `Invoke-RestMethod` 调一个返回 JSON 的接口，输出到文件：

```powershell
$resp = Invoke-RestMethod -Uri "https://api.example.com/data"
$resp | ConvertTo-Json -Depth 5 | Out-File "result.json"
```

用记事本或 VS Code 打开 `result.json`，发现中文全变成了 `?`。即使你在终端里看到的部分输出正确，文件仍然是坏的。

更深的问题在于，即使把 `Out-File` 换成 `Set-Content` 或重定向 `>`，乱码形式可能不同，但依旧不正确。这说明 **管道、文件输出和 HTTP 响应解析这三个环节可能各自引入了不同的编码假设**。

## 根因：三层编码断裂

### 1. HTTP 响应体的解码
`Invoke-WebRequest` 和 `Invoke-RestMethod` 在处理响应体时，默认会遵循响应的 `Content-Type` 头中的 `charset`。如果 API 正确返回了 `Content-Type: application/json; charset=utf-8`，那么解析后的字符串通常是正确的。**第一个陷阱**在于：很多内部 API 只返回 `application/json`，不指定 `charset`。此时 Windows PowerShell 5.1 会使用 **ISO-8859-1**（即 Windows-1252 的超集）来解码字节流，UTF-8 的中文字节序列被直接映射到该字符集的高字节区，于是产生“锟斤拷”或空白。

### 2. 控制台/ISE 的显示编码
即使内存中的字符串已是正确的 Unicode，`PowerShell.exe` 或 `powershell_ise.exe` 的控制台代码页可能仍然是 936（GBK）或 437。当字符串被渲染到终端窗口时，字体或代码页不支持对应字符，就会显示为方框或问号。但这只是显示问题，不影响重定向，然而它严重干扰排错——你根本不知道内存里到底是不是正确的数据。

### 3. 文件输出的默认编码
这才是杀伤力最大的环节。在 Windows PowerShell 5.1 中：
- `Out-File` 和重定向 `>` 默认使用 **UTF-16 LE**（Unicode）编码，且**带有 BOM**。
- `Set-Content` 默认使用 **ASCII**（实际是系统 ANSI 代码页，如 GBK），对非 ANSI 字符直接丢弃或变成 `?`。
- `ConvertTo-Json` 本身输出正确字符串，但只要一经过默认输出文件 cmdlet，编码就坏了。

所以，即便 HTTP 解码正确、内存中字符串正确，最终写到磁盘的 JSON 仍然会是坏的。

## 做法：一份能在 PS 5.1 / 7 都稳的工程写法

下面给出一个经过多次踩坑验证的稳健模式，适用于 Windows 环境，且同时兼容 PowerShell 5.1 和 PowerShell 7。

```powershell
$uri = "https://api.example.com/data"

# 强制将响应当作字节流自己用 UTF-8 解码
$response = Invoke-WebRequest -Uri $uri -UseBasicParsing
$utf8 = [System.Text.Encoding]::UTF8
$rawString = $utf8.GetString($response.RawContentStream.ToArray())

# 如果确认返回的是 JSON，可直接解析
$obj = $rawString | ConvertFrom-Json

# 写文件时显式指定 UTF-8 without BOM
$obj | ConvertTo-Json -Depth 10 -Compress | 
    ForEach-Object { [System.IO.File]::WriteAllText("$PWD/result.json", $_, $utf8) }
```

关键点说明：
- **绕过自动编码猜测**：通过 `RawContentStream` 拿到原始字节流，再用 `UTF8.GetString` 解码，彻底无视缺失的 charset 头。
- **`-UseBasicParsing`** 避免 IE 引擎干扰，在某些无桌面 Windows Server 上也是必需的。
- **文件写入用 `[System.IO.File]::WriteAllText`** 并直接传入 `$utf8` 对象，避免使用 PowerShell 的默认编码文件 cmdlet。这会产生**不带 BOM 的 UTF-8 文件**，与 Linux/macOS 工具链兼容，后续被 OpenClaw 或其他 Agent 读取时不会多出古怪的 `\uFEFF` 前缀。
- 如果你必须使用 `Out-File`，请加 `-Encoding utf8`，但注意该参数在 PS 5.1 中会写入**带 BOM 的 UTF-8**，下游可能不认。在 PowerShell 7 中 `-Encoding utf8NoBOM` 才是无 BOM，脚本跨版本时需额外判断。

如果你追求函数复用，可以封装一个通用安全获取函数：

```powershell
function Get-Utf8Api {
    param([string]$Uri)
    $resp = Invoke-WebRequest -Uri $Uri -UseBasicParsing
    $utf8 = [System.Text.Encoding]::UTF8
    return $utf8.GetString($resp.RawContentStream.ToArray())
}
```

所有需要中文 JSON 的模块直接调用 `Get-Utf8Api`，不要再信赖默认行为。

## 踩坑点：你以为没问题，其实只是“碰巧没坏”

1. **PowerShell 7 不救你**  
   pwsh.exe 默认输出编码改为 UTF-8，且 `Invoke-RestMethod` 内置了更现代的编码探测，但在缺失 charset 时仍可能回退到默认编码（通常是 UTF-8），这时多数场景正常，但少数边缘 API 仍可能因为响应头不规范而乱码。另外，如果你在 pwsh 中调用了 Windows PowerShell 5.1 的兼容性模块，编码问题会重新出现。所以**显式控制解码逻辑**才是长久之计。

2. **JSON 作为中间产物时的再次转码**  
   你的脚本可能只是从 API 拉数据，然后通过管道传给另一个 Python 脚本或 Node 进程。在跨进程通信时，PowerShell 的标准输出如果被宿主程序用非 UTF-8 模式读取，中文再坏一次。建议一律落盘为 UTF-8 文件，通过文件路径传递，而不是用管道传原文。

3. **`ConvertTo-Json` 的深度和转义**  
   `-Depth` 默认只有 2，多层嵌套的对象会被截断，虽然不是编码问题，但常伴随出现。中文在 `ConvertTo-Json` 中默认不会被转义为 `\uXXXX`，但如果某天你发现中文变成了 unicode 转义序列，检查是否在其他环境中被再次 `ConvertTo-Json` 或使用了 `-EscapeHandling`。

## 可复用建议

- **所有文件写入操作，统一使用 `[System.IO.File]::WriteAllText()`，并显式传入 `[System.Text.Encoding]::UTF8`**。这是成本最低且跨版本一致的方案。
- **放弃 `Out-File -Encoding utf8`，除非你确定下游容忍 BOM**。在微服务链中，BOM 极容易让 JSON 解析器报错。
- **编写自动检查脚本**，在 CI 或部署前验证输出 JSON 文件的第一个字节不是 `0xEF,0xBB,0xBF` 或 `0xFF,0xFE`。
- **在 Agent/MCP 插件的 README 中写死要求**：开发环境请使用 `[System.IO.File]::WriteAllText`，不要依赖 PowerShell 默认的文件编码。
- 如果你的自动化宿主环境一定是 Windows Server 且无法安装 pwsh 7，则必须将上述字节流解码方案作为硬性规范。

## 总结

PowerShell 的“多默认编码”是历史债，中文 JSON 被打坏的核心原因就是 HTTP 响应字节 → 字符串 → 文件这三个环节中的某一个或某几个，悄悄使用了非 UTF-8 的默认编码。方案很简单：**从字节流开始就用 UTF-8 解码，写文件时用 .NET 的文件 API 强制指定 UTF-8**。在 OpenClaw 这类需要将中文数据流通过多个 Agent 模块的系统中，一次编码失误就足以让整个链路的结果变得不可读。

下一次再看到“中文变问号”，不用怀疑字体，先去查原始字节和写入编码。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-27/a2d65d369b9fb7ed.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-27/dd2d63b825986d92.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-27/1db58b5ad340b638.png)

