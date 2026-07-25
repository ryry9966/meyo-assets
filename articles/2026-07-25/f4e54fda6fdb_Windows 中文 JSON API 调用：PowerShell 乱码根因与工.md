---
title: Windows 中文 JSON API 调用：PowerShell 乱码根因与工程化修复
feedId: 30383
source: 综合讨论
publishedAt: 2026-07-25
---

## 1. 背景：自动化里被“打坏”的中文

在 OpenClaw 的 Agent、MCP 工具或各类自动化脚本中，通过 PowerShell 调用返回中文 JSON 的 REST API 是非常常见的需求。例如：

- 从内部知识库获取中文文档摘要；
- 查询 CMDB 中的中文资产信息；
- 调用大模型接口，拿到中文回答；
- 抓取业务系统状态，页面内包含中文提示。

典型的做法是使用 `Invoke-RestMethod` 或 `Invoke-WebRequest`，然后用 `ConvertFrom-Json` 解析。但很多同学在 Windows 环境下会看到这样的结果：

```
name : ????
desc : ä½ å¥½
```

或者写入文件后，用其他工具打开是正常中文，但 PowerShell 控制台输出乱码；更严重的是，后续字符串处理时直接丢失了正确的 Unicode 字符，导致 Agent 判断错误、入库变成问号。

这并不是个例，也不是 Windows 的“特性”，而是 PowerShell 的编码处理与 API 常见实践之间存在一条不易觉察的沟壑。下面从根因开始，给出可复现、可工程化落地的修复方案。

## 2. 问题本质：字符集转换的三次错位

绝大多数现代 API 返回的 JSON 使用 **UTF-8 编码**，并且在 HTTP 响应头中会标注：
```
Content-Type: application/json; charset=utf-8
```

PowerShell 在处理这些响应时，数据会经过至少三个关键转换点，任何一个环节出错就会产生乱码。

### 错位一：响应流 → .NET 字符串

`Invoke-RestMethod` 和 `Invoke-WebRequest` 底层使用 .NET 的 `HttpClient` 或 `WebRequest`。当你不使用 `-OutFile` 而是直接获取内容时，**PowerShell 会根据响应头的 `charset` 选择编码**。但如果 API 没有正确声明 `charset`（例如只有 `Content-Type: application/json`），或者实现了老旧的 HTTP/1.0 风格，.NET 可能退回默认编码。

在 .NET Framework 4.x（Windows PowerShell 5.1 依赖的 CLR）中，默认退路是 **ISO-8859-1**；在 .NET Core（PowerShell 7）中默认退路是 **UTF-8**。但无论如何，如果字节流被错误解释，中文就已经被破坏。

### 错位二：.NET 字符串 → 控制台输出

当你要把字符串打印到控制台时，Windows 控制台主机（conhost）需要将 .NET 内部使用的 UTF-16 字符串转成控制台代码页对应的编码。中文 Windows 的默认控制台代码页通常是 **936 (GBK)**，而不是 65001 (UTF-8)。

如果你的字符串中包含了 GBK 不支持的字符（虽然中文大多支持，但某些生僻字或 emoji 可能出问题），或者代码页被意外改成英文 437，你看到的就会是问号或乱码。注意：即使代码页是 936，而你强制向控制台输出 UTF-8 字节流，也会因为双重编码导致乱码，因为控制台再用 GBK 去解释 UTF-8 字节。

### 错位三：文件写入的默认编码

`Out-File`、`Set-Content`、`>` 重定向在 Windows PowerShell 5.1 中默认使用 **UTF-16 LE** 或 **ASCII**（取决于是否检测到宽字符），并且会写 BOM。`Invoke-WebRequest` 不指定 `-OutFile` 时，内部 `.Content` 属性已经是字符串，再用 `Out-File` 保存可能引入新的编码转换问题。不少同学发现直接 `-OutFile` 保存下来的文件是好的，但转成字符串再写文件就坏了，其原因就在于手动操作时破坏了原始字节。

## 3. 可复现的踩坑演示

假设一个最简单的测试 API 返回：
```json
{"message": "你好，世界！"}
```
使用 Windows PowerShell 5.1 默认配置，执行：
```powershell
$resp = Invoke-RestMethod -Uri "http://127.0.0.1:5000/api/test"
$resp.message
```
控制台输出正常，因为响应头含 `charset=utf-8`，且控制台代码页是 936（兼容 GBK），中文可以正常显示。但一旦我们做进一步加工：
```powershell
$resp.message | Out-File output.txt
```
用 `type output.txt` 查看可能正常，因为 `Out-File` 默认 UTF-16，再用 `type` 读回时 PowerShell 会自动识别编码。但如果用其他工具（如记事本）打开看到乱码，或者脚本将此文件交给一个只接受 UTF-8 的下游系统，就会出问题。

更隐蔽的情况：API 返回的 JSON 中嵌入了 base64 编码的字节流，我们解码后得到的是 UTF-8 字节，但直接用 `[System.Text.Encoding]::Default.GetString()` 而不是 `UTF8.GetString()`，也会乱码。

另外，不少同学曾试图通过 `chcp 65001` 和设置 `[Console]::OutputEncoding` 来解决，却发现 `Invoke-RestMethod` 返回的内容依然是乱码，这通常说明破坏发生在步骤 1（字节→字符串），字符串已经损坏，控制台只能显示已经被损坏的内容。

## 4. 工程化修复方案（分层防御）

下面是一套从根上解决问题的方案，按推荐程度排列。

### 4.1 直接操作原始字节（最稳妥）

```powershell
$response = Invoke-WebRequest -Uri $uri -Method Get -ContentType "application/json; charset=utf-8"
# 从 Content 获取的已经是字符串，可能已被损坏。改用 RawContentStream
$stream = $response.RawContentStream
$reader = [System.IO.StreamReader]::new($stream, [System.Text.Encoding]::UTF8)
$jsonString = $reader.ReadToEnd()
$reader.Close()

$obj = $jsonString | ConvertFrom-Json
```

或者更简洁，使用 `Invoke-WebRequest` 的 `-UseBasicParsing` 并直接保存文件，再从文件读回：
```powershell
Invoke-WebRequest -Uri $uri -OutFile temp.json
$obj = Get-Content temp.json -Raw -Encoding UTF8 | ConvertFrom-Json
```
这样可以完全绕过 PowerShell 的自动编码推断，保证原始字节被 UTF-8 正确解码。

### 4.2 设置会话级编码参数（适用于保留对象结构）

如果必须使用 `Invoke-RestMethod` 来直接获取对象（因为它会自动反序列化），可以在脚本开头强制指定编码：
```powershell
# PowerShell 7+
$PSDefaultParameterValues['Invoke-RestMethod:ContentType'] = 'application/json; charset=utf-8'
$PSDefaultParameterValues['Out-File:Encoding'] = 'utf8'
$OutputEncoding = [System.Text.UTF8Encoding]::new($false)  # 无 BOM
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
```

对于 Windows PowerShell 5.1，`$OutputEncoding` 的设置很重要，因为很多 cmdlet 在向控制台输出或管道传送时使用此编码。但注意，它主要影响**输出**，不能修复已损坏的内部字符串。因此仍需要配合 4.1 确保字符串本身正确。

### 4.3 修复控制台显示（仅用于开发调试）

```powershell
chcp 65001 > $null
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
```
同时在终端属性中勾选“使用 Unicode UTF-8 提供全球语言支持”（Windows 10 1903+ 可选功能，但可能影响其他旧程序）。这能保证字符串正确显示，但不能解决字符串内容已经变成“???”的问题。调试时结合 4.1 使用。

### 4.4 封装可复用的安全函数

将上述逻辑封装成 OpenClaw 插件或 MCP 工具可调用的函数，如下所示：
```powershell
function Invoke-Utf8RestMethod {
    param(
        [string]$Uri,
        [string]$Method = 'Get',
        $Body
    )
    $tmpFile = [System.IO.Path]::GetTempFileName()
    try {
        $params = @{
            Uri     = $Uri
            Method  = $Method
            OutFile = $tmpFile
            ContentType = "application/json; charset=utf-8"
        }
        if ($Body) { $params['Body'] = ($Body | ConvertTo-Json -Depth 10) }
        Invoke-WebRequest @params | Out-Null
        $json = Get-Content $tmpFile -Raw -Encoding UTF8
        return $json | ConvertFrom-Json
    } finally {
        Remove-Item $tmpFile -Force -ErrorAction SilentlyContinue
    }
}
```
这样在所有自动化流程中统一使用此函数，彻底规避编码问题。

## 5. 可复用建议总结

- **永远不要假设默认编码**：Windows PowerShell 5.1 的默认编码行为复杂，在服务器、任务计划、非交互式环境下尤其容易出问题。
- **原始字节流优先**：对于 JSON API，尽量用文件中介或流读取，显式指定 UTF-8 解码。
- **避免用字符串变换代替文件操作**：`Invoke-WebRequest` 的 `.Content` 属性可能已损坏，如果要保存，直接用 `-OutFile`。
- **统一团队环境**：在 OpenClaw 的 Agent 初始化脚本中，强制设置 `$OutputEncoding` 和 `[Console]::OutputEncoding`，并将代码页设为 65001。
- **测试你的 API 响应头**：确保后端在 Content-Type 中准确写明 `charset=utf-8`，从源头减少猜测。

## 6. 总结

PowerShell 中文乱码的根源在于：**字节→字符串→控制台/文件**这三层转换中，编码信息没有正确传递。它既不是 PowerShell 的缺陷，也不是 Windows 的原罪，而是历史兼容性与现代 UTF-8 实践之间的鸿沟。在 OpenClaw、Agent 和 MCP 这类高度依赖自动化串联的工具链中，一个乱码可能引发级联故障：Agent 读到乱码后将其写入数据库，后续查询全部失败；MCP 插件返回乱码给主控，导致大模型产生幻觉式回复。

通过直接处理原始字节流、显式声明 UTF-8 编码、封装可复用函数，我们可以把这条鸿沟填平，让中文 JSON 数据在 Windows 自动化流程中畅通无阻。

记住一句话：**在 PowerShell 里处理中文，永远把编码握在自己手里，不要交给隐式转换。**

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-25/3bb0f2e4c01a188c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-25/f67e6bfde7de2c45.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-25/8cdf565a77630530.png)

