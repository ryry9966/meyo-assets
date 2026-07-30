---
title: PowerShell 中文 JSON API 调用的编码深坑：返回的文本为什么全是乱码
feedId: 31008
source: 综合讨论
publishedAt: 2026-07-30
---

# PowerShell 中文 JSON API 调用的编码深坑：返回的文本为什么全是乱码

在 Windows 上写自动化脚本，用 PowerShell 调用一个返回中文内容的 JSON API，然后传给下游的 Agent 或 MCP 插件，结果预览时满屏的“锟斤拷”“烫烫烫”或者“????”，这种事你一定不陌生。网上搜一堆“设置编码”的碎片答案，但很多时候按着做了还是打坏，因为坑不止一个。本文把 Windows PowerShell 处理中文 JSON 时最容易踩的 3 类编码错位讲清楚，并给出真正能在工程里复用的解决思路。

---

## 背景：自动化流水线里的中文字符旅行

假设你有一个本地或远程的 REST API，返回体类似：

```json
{
  "title": "生产环境中 Agent 异常排障记录",
  "summary": "今日巡检发现 OpenClaw 插件在解析 MCP 响应时超时……"
}
```

你通过 `Invoke-RestMethod` 获取数据，然后把 `title` 或 `summary` 写入文件、拼装成下一条请求，或者输出到控制台交给下一个工具。链路一旦有一环用了错误的编码假设，中文就会变成不可读的字符。

最常见的触发场景：
- 在标准 Windows PowerShell（5.1）里运行脚本
- API 返回的 `Content-Type` 是 `application/json; charset=utf-8`
- 控制台编码不是 UTF-8，或者文件写入时用了默认的 Unicode（UTF-16 LE）
- 使用了 `Out-File`、`>` 重定向，但没有显式指定编码

---

## 为什么 PowerShell 会把中文打坏？

### 核心原因：流解码与终端编码两层错位

**第一层：HTTP 响应流的解码错误**

`Invoke-RestMethod` 和 `Invoke-WebRequest` 内部会尝试自动检测响应的编码。如果服务器返回了明确的 `charset=utf-8`，理论上 PowerShell 应该正确解码。但问题在于 **Windows PowerShell 5.1 在自动解码时，有时会忽略响应头的 charset 而使用 ISO-8859-1 回退**，尤其是当响应没有 BOM 且 `Content-Type` 缺失或不够标准时。这就导致 UTF-8 字节流被按单字节扩展 ASCII 错误解释，中文变成一堆乱码。

**第二层：控制台输出编码**

即使 PowerShell 内部已经正确解码成了 .NET 的字符串（UTF-16 内存表示），当你把它打印到控制台时，控制台宿主（conhost 或 Windows Terminal）需要将字符串转成当前代码页下的字节。**中文 Windows 控制台默认代码页是 936（GBK）**，而 PowerShell 输出的编码默认跟随 `[Console]::OutputEncoding`，这个值在多数系统上也是 GBK。如果字符串中有 GBK 无法表示的罕见字，或者你之前通过 `chcp 65001` 切到 UTF-8 但 `[Console]::OutputEncoding` 没有同步修改，显示还是会乱。

**第三层：文件写入的 BOM 与编码不匹配**

很多人习惯 `$result | Out-File result.json` 或者 `$result > result.json`。在 Windows PowerShell 中，`Out-File` 的默认编码是 **UTF-16 LE with BOM**，而 `>` 重定向底层用的 `Out-File`，所以文件会变成双字节编码，任何后续按 UTF-8 读取的工具都会完全挂掉。即使你用了 `-Encoding UTF8`，PowerShell 5.1 的 `-Encoding UTF8` 仍然会在文件头加上 BOM，而很多后端服务或命令行解析器并不接受 BOM。

三层叠加，中文一路被打坏。

---

## 做法 / 步骤：从抓取到存储的 UTF-8 闭环

下面给出一组可复用的脚本模板，同时兼容 Windows PowerShell 5.1 和 PowerShell 7+。

### 1. 正确获取 API 响应并强制 UTF-8 解码

```powershell
# 推荐：使用 Invoke-WebRequest 获取原始字节，再按 UTF-8 转字符串
$response = Invoke-WebRequest -Uri "https://api.example.com/data" -UseBasicParsing
$rawBytes = $response.Content
# 如果 $rawBytes 已经是字符串，则直接按正确编码重新解码
$utf8String = [System.Text.Encoding]::UTF8.GetString(
    [System.Text.Encoding]::Default.GetBytes($rawBytes)
)
$obj = $utf8String | ConvertFrom-Json
```

如果确定服务端始终返回 UTF-8，也可以在脚本顶部设置全局变量，影响部分 cmdlet 的默认行为：

```powershell
$PSDefaultParameterValues['*:Encoding'] = 'utf8'
```

### 2. 让控制台能正确显示中文

```powershell
# 同时设置控制台代码页和 PowerShell 输出编码
chcp 65001 > $null
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
```

如果你习惯在 VS Code 内建终端或 Windows Terminal 里跑脚本，这一步几乎必做。

### 3. 写入文件时直接使用字节流或明确指定无 BOM UTF-8

```powershell
# 方案 A：Set-Content（PowerShell 5.1 支持 -Encoding UTF8NoBOM 需要 PS 6+，5.1 只能用 UTF8 会带 BOM）
# 在 PS 5.1 中，绕过 BOM 的稳妥方式是直接写字节
[System.IO.File]::WriteAllText(
    "result.json",
    $utf8String,
    [System.Text.UTF8Encoding]::new($false)  # $false 表示无 BOM
)
```

或者升级到 PowerShell 7，然后无脑：

```powershell
$utf8String | Set-Content -Path result.json -Encoding UTF8NoBOM
```

### 4. 验证字节

快速检查文件前几个字节，确认没有 BOM 且是有效 UTF-8：

```powershell
Format-Hex -Path result.json -Count 32
```

---

## 踩坑点汇总

1. **“我在 ISE 里看是好的，但命令行就跑乱”**  
   ISE 有自己的文本渲染环境，与控制台宿主不一致，不要作为编码正确的依据。

2. **ConvertTo-Json 把中文转成了 `\uXXXX`**  
   这是 `ConvertTo-Json` 的默认转义行为，符合 JSON 标准。下游在 `ConvertFrom-Json` 时会自动还原成中文，无需恐慌。但如果需要直接肉眼可读的 JSON 文件，可以在 PowerShell 7 中使用 `-EscapeHandling EscapeNonAscii` 参数控制。

3. **管道写入数据库或 MCP 工具被打回**  
   许多本地 Agent 或 MCP 实现基于 Node.js/Python，默认期待 UTF-8 无 BOM。从 PowerShell 传数据时，务必将内存中的字符串用 UTF-8 编码后传送，不要依赖 `Out-String` 的默认行为。

4. **文件路径含中文导致莫名其妙失败**  
   文件写入编码问题通常会连锁影响路径解析，建议全部路径变量也用 UTF-8 管理，或者直接用英文命名。

---

## 可复用建议

- **统一使用 PowerShell 7**：它的 `Encoding` 参数支持更丰富的选项，且默认输出、文件写入都倾向于 UTF-8 无 BOM，大幅减少幻觉乱码。
- **在所有自动化脚本头部固定三行**：
  ```powershell
  $OutputEncoding = [System.Text.Encoding]::UTF8
  [Console]::OutputEncoding = [System.Text.Encoding]::UTF8
  $PSDefaultParameterValues['*:Encoding'] = 'utf8'
  ```
- **API 调用优先看响应头**：使用 `-ResponseHeadersVariable` 捕获头信息，打印 `Content-Type`，确认 charset。
- **跨进程传递 JSON 时，直接通过管道传字节流**：`[System.Text.Encoding]::UTF8.GetBytes($jsonString)` 扔给下游，杜绝字符串隐式转码。
- **养成用 `Test-Json` 和 `Format-Hex` 做快速验证的习惯**，不要靠“看起来正常”来判断。

---

## 总结

PowerShell 打坏中文 JSON，根源不是 Bug，而是 Windows 历史遗留的多编码环境与 PowerShell 默认行为之间的错配。在 OpenClaw / Agent / MCP 这类高度自动化的工具链里，任何一处编码假设不一致都会造成数据不可读，进而让自动化流程“静默”失败。把这个“文字传输协议”固定为 UTF-8 无 BOM，是让整个中文处理管线可靠运转的第一步，也是最容易忽视的一步。

---

