---
title: PowerShell 打坏中文 JSON：Windows 编码迷局与工程化解法
feedId: 29264
source: 综合讨论
publishedAt: 2026-07-16
---

## 背景：自动化脚本里的「锟斤拷」

在 OpenClaw 社区，不少朋友会在 Windows 上用 PowerShell 构建自动化工具或 Agent 插件。比如调用一个大模型的 API，得到一段包含中文的 JSON 响应，然后用 PowerShell 的 `Invoke-RestMethod` 直接解析，结果在控制台输出或写入文件时，中文变成了 `????`、`锻炼手指` 之类的乱码，甚至直接抛异常说 JSON 格式无效。

这个问题在纯英文环境里很少出现，但在国内开发者日常的中文 API 交互中几乎是必踩的坑。而且它不是单一原因，而是 PowerShell 的输入输出编码、.NET 底层默认设置、以及 Cmdlet 内部解码逻辑共同作用的「系统性偏差」。

## 问题精确定位：到底哪里打坏了中文？

先做一个最常见的场景还原：

```powershell
$response = Invoke-RestMethod -Uri "https://api.example.com/zh-message" -Method Get
$response.message
```

API 返回的 JSON 类似：

```json
{"code":0,"message":"操作成功"}
```

但在 PowerShell 控制台看到的是 `操作??` 甚至无法解析。如果你把 `$response` 再用 `ConvertTo-Json` 输出到文件：

```powershell
$response | ConvertTo-Json | Out-File result.json
```

打开文件里面可能是 `"\u64cd\u4f5c\u6210\u529f"` 或者一堆问号。

**关键链条有三个环节会出问题：**

1. **响应字节流的解码** —— `Invoke-RestMethod` 默认如何将 HTTP 响应的字节转换成字符串？
2. **控制台/宿主显示编码** —— 字符串对象在控制台输出时的编码转换。
3. **文件输出编码** —— `Out-File` 或重定向 `>` 使用的默认编码。

这三个环节的默认行为，在英文 Windows 中通常被设定为系统 ANSI 代码页（如 Windows-1252），对中文极不友好。

## 深入一层：Invoke-RestMethod 的隐蔽解码逻辑

许多人会尝试在调用时指定 `-ContentType "application/json; charset=utf-8"`，但 `-ContentType` 其实是设置**请求头**的 Content-Type，并不影响对响应内容的解码。`Invoke-RestMethod` 处理响应编码的流程如下：

1. 读取响应头里的 `Content-Type`，如果包含 `charset=xxx`，则使用该编码。
2. 如果没有显式指定 charset，或响应头缺失，则默认采用 **ISO-8859-1**（Latin-1）编码。
3. 将解码后的字符串交给 JSON 解析器。

问题来了：很多国内 API 虽然返回 UTF-8 编码的 JSON，但响应头里可能只写 `Content-Type: application/json`，**没有 charset**。这时候 PowerShell 就按 ISO-8859-1 去解码 UTF-8 字节流，中文自然会变成两字节乱码，甚至因为非法 UTF-8 序列导致 `ConvertFrom-Json` 直接报错。

> 这是最容易掉进去的坑：即使 API 本身没问题，只要响应头里偷懒没写 `charset=utf-8`，你的脚本就大概率要挂。

## 正确的做法：从字节流接管解码权

放弃了让 `Invoke-RestMethod` 自动处理，改为用 `Invoke-WebRequest` 获取原始字节流，然后手动按 UTF-8 解码。

```powershell
$response = Invoke-WebRequest -Uri "https://api.example.com/zh-message" -Method Get
# $response.Content 此时已根据响应头自动解码，可能已损坏
# 所以必须取 RawContentStream
$stream = $response.RawContentStream
$reader = [System.IO.StreamReader]::new($stream, [System.Text.Encoding]::UTF8)
$jsonString = $reader.ReadToEnd()
$data = $jsonString | ConvertFrom-Json
$data.message
```

这样绕过了一切自动编码探测，强制用 UTF-8 解析。如果 API 的编码很确定（比如 UTF-8），这是最稳健的工程化方法。

控制台输出前，还需要保证宿主编码正确：

```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$OutputEncoding = [System.Text.Encoding]::UTF8
```

这两行放在脚本开头，可以解决大部分控制台中文输出问题。

## 文件输出与管道陷阱

即使内存里的字符串是完好的，当你使用 `> result.txt` 或 `Out-File` 时，中文也可能再次受损。原因在于：

- PowerShell 的 `>` 实际上是 `Out-File` 的语法糖，默认使用 **Unicode (UTF-16LE)** 或 **ASCII**，具体取决于 PowerShell 版本和系统配置。
- `Out-File` 在没有 `-Encoding` 参数时，经常退化为系统 ANSI 编码（非 UTF-8）。
- `ConvertTo-Json` 默认会将非 ASCII 字符转义为 `\uXXXX`，这是为了 JSON 安全，但可读性差。如果希望保留中文，需添加 `-Compress` 并用 `-EscapeHandling EscapeNonAscii` 配合（PowerShell 7+），不过更简单的办法依然是控制输出编码。

稳妥地写出中文 JSON 文件：

```powershell
$data | ConvertTo-Json -Depth 5 | Out-File -Encoding utf8 "result.json"
```

或者使用 `Set-Content`：

```powershell
$data | ConvertTo-Json | Set-Content -Encoding utf8 "result.json"
```

另外，避免用 `>` 重定向输出 JSON 到文件，除非你已经通过 `$PSDefaultParameterValues` 锁定了默认编码：

```powershell
$PSDefaultParameterValues['Out-File:Encoding'] = 'utf8'
```

## 踩坑点小结

1. **缺 charset 响应头**：很多 API 不标明 charset，导致 `Invoke-RestMethod` 按 ISO-8859-1 解码，中文损坏。
2. **控制台无声丢失**：内存中字符串正常，但控制台显示问号，因为 `[Console]::OutputEncoding` 仍是 GBK 或 ASCII。
3. **文件输出变非 UTF-8**：直接用 `>` 或 `Out-File` 写入文件，因默认编码造成乱码或带 BOM 问题。
4. **JSON 转义后“假乱码”**：`ConvertTo-Json` 将中文转义为 `\uXXXX`，这不是乱码，但可读性差，容易让新手误判为编码问题。
5. **BOM 干扰**：用某些编辑器创建 PowerShell 脚本时带着 UTF-8 BOM，可能导致奇怪解析错误；推荐在脚本保存时选择“UTF-8 without BOM”。

## 可复用的工程建议

给你的自动化脚本加一个标准的“环境初始化”块：

```powershell
# 确保整个会话使用 UTF-8
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$OutputEncoding = [System.Text.Encoding]::UTF8
$PSDefaultParameterValues['Out-File:Encoding'] = 'utf8'
$PSDefaultParameterValues['Set-Content:Encoding'] = 'utf8'

# 强制 Invoke-WebRequest 使用 UTF-8 解码（当响应头不可靠时）
function Invoke-WebRequestUtf8 {
    param($Uri, $Method = 'Get')
    $resp = Invoke-WebRequest -Uri $Uri -Method $Method -UseBasicParsing
    $reader = [System.IO.StreamReader]::new($resp.RawContentStream, [System.Text.Encoding]::UTF8)
    return $reader.ReadToEnd() | ConvertFrom-Json
}
```

之后你的 Agent 或插件只需统一调用这个包装函数，就可以一劳永逸地避开绝大多数中文编码问题。

如果你构建的是 OpenClaw 的 PowerShell 插件，建议在模块的 `.psm1` 文件顶部就加上上述编码初始化代码，并在所有对外 API 调用中统一使用 `Invoke-WebRequestUtf8` 或等效的字节流解码。

## 总结

Windows 上的 PowerShell 在处理中文 JSON 时确实有「性格」——它继承了很多 .NET 默认值和历史包袱。但这些问题本质上都源于**编码在整个处理链上不透明**。一旦你明白字节→字符串→JSON 对象的每一步编码决定权在哪里，就可以很有针对性地接管控制权：

- 从网络上拿到的是字节流，别让 Cmdlet 猜编码，直接指定 UTF-8。
- 输出到文件时，显式指定 `-Encoding utf8`。
- 在脚本开头设置控制台编码，让调试信息也清晰可见。

这些操作看起来繁琐，但都是可以封装成一次性的工程动作。当你的 Agent 插件在 Windows 服务器上稳定运行、中文日志不乱码、API 返回值不经转义时，你会发现这些几分钟的深入理解非常值得。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-16/f2640f22c9786188.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-16/b6acca101e83f5a6.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-16/6c195c9f2c9ca059.png)

