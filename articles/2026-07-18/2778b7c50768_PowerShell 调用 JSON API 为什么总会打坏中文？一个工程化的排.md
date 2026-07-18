---
title: PowerShell 调用 JSON API 为什么总会打坏中文？一个工程化的排查与修复指南
feedId: 29558
source: 综合讨论
publishedAt: 2026-07-18
---

## 背景：一个让人抓狂的“小问题”

在 Windows 上用 PowerShell 写 Agent、MCP 连接器或自动化脚本时，你大概率会碰到这样的场景：

- 明明 API 返回的 JSON 里中文正常，PoSH 输出到控制台却变成 `????` 或不知所云的符号。
- 把结果 `> result.json` 后，文件里的中文变成了 `\uXXXX` 转义序列，下游解析器报错。
- `ConvertTo-Json` 一下，所有非 ASCII 字符全部被转义，跟服务端交互时出现数据不一致。

这个问题表面看是“编码”，实际牵扯到 Windows 生态里 PowerShell 的默认行为、控制台宿主、.NET 运行时对字符串的处理，以及管道重定向的编码假设。对于依赖 JSON 的 Agent 流程来说，这种“静默损坏”远比报错更可怕，因为在调试看到乱码之前，你可能已经用错误的数据做了一轮决策。

下面按 **定位 -> 复现 -> 修复 -> 封装** 的顺序，给你一套可直接复用的解决方案。

## 问题在哪里：三层编码陷阱

### 1. 输出编码（控制台 / 重定向）
Windows PowerShell 5.1 的默认输出编码是 ASCII（SBCS），具体由系统区域设置决定（如代码页 936 简体中文）。当你把对象或字符串直接管道重定向到文件 (`> file.txt`)，它默认用 **UTF-16 LE** 写入（实际上是 Unicode 编码），但控制台显示却受 `[Console]::OutputEncoding` 控制。如果你在中文 Windows 上，`[Console]::OutputEncoding` 通常是 `GB2312`，但很多 API 返回的是 UTF-8 字节流，PoSH 自动解码时若未明确声明，就会按系统代码页错误解释，造成中文字符损坏。

### 2. JSON 序列化的“不安全”默认行为
`ConvertTo-Json` 在 **Windows PowerShell 5.1** 中默认会把所有非 ASCII 字符转义为 `\uNNNN`，这是 ECMA-404 允许的行为，但对中文极不友好。例如：
```powershell
@{ name = '你好' } | ConvertTo-Json
# 输出：{ "name": "\u4f60\u597d" }
```
下游如果是强类型反序列化器（如 Go 的 `encoding/json`）一般能正确处理；但如果你的流程里存在“字符串拼接、匹配、日志告警”这类未经反序列化的环节，中文就变成了一串十六进制。更糟糕的是，如果你把这个 JSON 作为 Body 再发给另一个服务，而那个服务没有正确解码 Unicode 转义，就会产生脏数据。

### 3. Web cmdlet 的隐式解码
`Invoke-RestMethod` / `Invoke-WebRequest` 返回的 `Content` 属性会尝试根据响应的 `Content-Type` 头自动解码。但很多国产 API 并不规范（`charset` 缺失或给了 `GBK`），加上 Windows 上 .NET Framework 的 `HttpWebRequest` 对压缩、BOM 的处理有时会出错，导致中文乱码在内存中已经发生，后续做任何编码转换都是徒劳。

## 从复现到修复：可落地的步骤

### 复现环境
- Windows 10 / 11，PowerShell 5.1
- 目标 API：一个返回 `{ "msg": "操作成功" }` 的 HTTP 服务

**复现脚本**：
```powershell
$r = Invoke-RestMethod -Uri 'http://example.com/api/test' -Method Get
$r.msg   # 控制台可能显示“操作??”
$r | ConvertTo-Json | Out-File -FilePath result.json
# result.json 中 msg 变成了 "\u64cd\u4f5c\u6210\u529f"
```

### 修复三步走

#### 第一步：控制输入流的解码
放弃依赖 `Invoke-RestMethod` 的自动解码，显式读取字节流并用 UTF-8 解码：
```powershell
$resp = Invoke-WebRequest -Uri '...' -Method Get -UseBasicParsing
$rawBytes = $resp.RawContentStream.ToArray()
$jsonString = [System.Text.Encoding]::UTF8.GetString($rawBytes)
$obj = $jsonString | ConvertFrom-Json
```
如果 API 返回的是 GBK/GB2312，则替换 `UTF8` 为 `GetEncoding('GBK')`。一个更稳妥的做法是先通过响应头 `Content-Type` 抽取出 charset，再选择编码器。

#### 第二步：安全输出到文件和控制台
为避免重定向乱码，放弃 `>`，使用 `Out-File` 或 `Set-Content` 并显式指定编码：
```powershell
# 输出到文件（UTF-8 无 BOM，兼容性最好）
$utf8NoBom = New-Object System.Text.UTF8Encoding $false
[System.IO.File]::WriteAllText('result.json', $jsonString, $utf8NoBom)

# 或者用 Out-File
$jsonString | Out-File -FilePath result.json -Encoding utf8NoBOM
```
此时控制台若仍显示异常，检查 `[Console]::OutputEncoding`：
```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
```

#### 第三步：让 ConvertTo-Json 保留中文
**PowerShell 6.2+** 支持 `-EscapeHandling` 参数：
```powershell
$obj | ConvertTo-Json -EscapeHandling EscapeNonAscii
```
但在 **Windows PowerShell 5.1** 中没有这个参数，只能“曲线救国”：
- 使用 .NET 的 `System.Web.Script.Serialization.JavaScriptSerializer`（已过时，但能用）
- 或直接用 `Newtonsoft.Json`（如果已安装）：
  ```powershell
  [Newtonsoft.Json.JsonConvert]::SerializeObject($obj, [Newtonsoft.Json.Formatting]::None)
  ```
- 如果不想引入外部库，可以利用 `ConvertTo-Json` 生成字符串后，再用正则将 `\uXXXX` 替换为实际字符：
  ```powershell
  $json = $obj | ConvertTo-Json -Depth 10
  [Regex]::Replace($json, '\\u([0-9A-Fa-f]{4})', {
      param($m)
      [char][int]::Parse($m.Groups[1].Value, 'HexNumber')
  })
  ```
  这个正则法简单有效，适合很多 Agent 脚本场合。

## 踩坑点与防掉进坑里的建议

1. **编码传染性**：一个环节的错误编码会污染整个管道。比如你从文件读取 JSON 时没有指定 `-Encoding`，`Get-Content` 默认在 PS5.1 下是 ASCII（可能丢中文），PS6+ 才是 UTF-8。永远显式指定 `-Encoding UTF8`。
2. **BOM 的地雷**：Windows 上的文本编辑器（比如记事本）喜欢加 BOM。如果你生成的 JSON 文件带了 BOM，API 服务器可能解析失败，返回莫名其妙的错误。使用 `utf8NoBOM` 输出是唯一安全的选择。
3. **MCP / stdio 通信**：当你用 PowerShell 实现 MCP 服务器时，与客户端（如 Claude Desktop）的通信是通过 stdin/stdout 进行的，必须以 UTF-8 字节流形式交互。在脚本开头强制设置：
   ```powershell
   $OutputEncoding = [System.Text.UTF8Encoding]::new($false)
   $PSDefaultParameterValues['*:Encoding'] = 'utf8'
   ```
4. **跨版本差异**：PowerShell 7+ 已大大改善这些行为，但很多企业 Windows 环境仍锁定在 5.1。如果你同时为两个版本写脚本，需要做兼容处理（如检查 `$PSVersionTable.PSVersion` 来决定是否可用 `-EscapeHandling`）。

## 可复用封装：一个“安全 JSON 管道”函数

在你的 Agent 工具箱里可以放这样一个函数，一次性解决 90% 的中文编码问题：

```powershell
function Invoke-SafeJsonApi {
    param($Uri, $Method = 'Get', $Body)
    $resp = Invoke-WebRequest -Uri $Uri -Method $Method -Body $Body -UseBasicParsing
    $charset = if ($resp.Headers['Content-Type'] -match 'charset=([\w-]+)') { $matches[1] } else { 'utf-8' }
    $enc = [System.Text.Encoding]::GetEncoding($charset)
    $jsonStr = $enc.GetString($resp.RawContentStream.ToArray())
    
    # 可选：修复 Unicode 转义以便人类阅读
    if ($jsonStr -match '\\u[0-9A-Fa-f]{4}') {
        $jsonStr = [Regex]::Replace($jsonStr, '\\u([0-9A-Fa-f]{4})', {
            [char][int]::Parse($args[0].Groups[1].Value, 'HexNumber')
        })
    }
    return $jsonStr | ConvertFrom-Json
}
```

使用它来处理所有外部 API 调用，你就无需在业务逻辑里反复纠结编码问题。

## 总结

PowerShell 把中文打坏，本质上是“自动解码/编码”在 Windows 遗留编码环境下的错配。解决的核心是四个字：**显式编码**——显式声明输入流的解码方式，显式指定输出文件和管道的编码，显式控制 JSON 序列化的转义行为。

作为一个 Agent 开发者，你肯定不希望自己的自动化链条因为一个字节的编码错误而做出错误判断。把上面这套方案沉淀到你的基础设施脚本里，远比每次出现乱码再去 Google “PowerShell 中文乱码” 要划算得多。

---

