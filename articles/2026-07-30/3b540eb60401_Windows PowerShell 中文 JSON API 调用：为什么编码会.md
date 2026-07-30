---
title: Windows PowerShell 中文 JSON API 调用：为什么编码会“吃掉”你的中文，以及如何修好它
feedId: 31017
source: 综合讨论
publishedAt: 2026-07-30
---

## 背景：自动化脚本里的中文去哪儿了

在 OpenClaw、Agent、MCP 插件这类工程中，几乎每天都要用脚本调用各种 API：OpenAI 接口、自建工具、数据管道。Windows 上的工程师习惯用 PowerShell 写胶水代码——`Invoke-RestMethod` 一行就能拿到 JSON，再交给下游处理。可一旦 API 返回的内容包含中文，就会碰到一个幽灵般的问题：

```
{
  "content": "浣犲ソ涓栫晫"
}
```

看着像乱码，又不像纯粹的 UTF-8 被误读为 Latin-1 的样子。更离奇的是，同样的字符串在控制台 `Write-Host` 是正常的，但只要输出到文件、赋值给变量然后导出日志，中文就坏了。这背后是 PowerShell 在 Windows 上处理管道、编码与 BOM 的一系列设计决策，几乎每个做自动化的人都会踩一次。

## 问题根源：三个关键机制让中文“碎掉”

要理解乱码，只需要抓住三件事：

1. **PowerShell 5.1（Windows 自带）的默认输出编码不是 UTF-8**  
   在 Windows PowerShell 5.1 里，`>` 和 `Out-File` 默认使用 **UTF-16 LE**（Unicode）编码。这是因为 PowerShell 内部字符串就是 .NET 的 `System.String`，采用 UTF-16 存储。当你不指定编码输出时，它理所当然地用 UTF-16 写文件。而那些期望 UTF-8 的工具（比如日志分析、编辑器、后续脚本）就会把每个双字节当作独立字符解读，产生“浣犲ソ”这类乱码。

2. **管道与重定向的编码由 `$OutputEncoding` 控制，而它的默认值又是 ASCII**  
   很多自动化会通过管道将 JSON 传递给外部程序（如 `curl`、`jq`、Python 脚本）。但 PS 5.1 的 `$OutputEncoding` 默认是 `US-ASCII`！这意味着从 PowerShell 管道发出的字符串会被强制按 ASCII 编码发出，所有非 ASCII 字符（中文）直接被替换成 `?` 或者高位字节截断。更要命的是，当外部程序读取标准输入时，看到的可能已经是残废数据了。

3. **Invoke-RestMethod 的 Content-Type 陷阱**  
   `Invoke-RestMethod` 会根据响应的 `Content-Type` 自动解码字节流。服务器如果返回 `Content-Type: application/json; charset=utf-8`，它一般会正确处理。但如果 API 没有明确指定 charset，或者返回了 `charset=gb2312`（一些老旧系统），PowerShell 可能会用 .NET 默认编码解析，造成乱码。另外，当使用 `-OutFile` 保存到文件时，编码又回到了问题一，乱码再生。

这三个机制叠加，让“控制台看着正常、文件/管道里却乱码”成了高频故障。

## 正确的做法：显式控制每一次编解码

### 1. 正确接收 API 响应，确保内存中是正确字符串

如果 API 明确返回 UTF-8 JSON，最安全的方式是直接拿到字节流，用 UTF-8 解码，避免自动推断。

```powershell
$response = Invoke-WebRequest -Uri "https://api.example.com/data" -UseBasicParsing
$rawBytes = $response.RawContentStream.ToArray()
$jsonString = [System.Text.Encoding]::UTF8.GetString($rawBytes)   # 强行按 UTF-8 解释
$obj = $jsonString | ConvertFrom-Json
```

如果 Content-Type 标头可信，也可以用 `Invoke-RestMethod` 直接解析成对象，但务必在事后验证：`$obj.content` 是否呈现正确中文。若不可信，就回到字节流方案。

### 2. 保存文件时，永远指定 UTF-8（不带 BOM 或带 BOM，看下游需求）

```powershell
# 写入 UTF-8 无 BOM（绝大多数 Linux 工具喜欢）
$jsonString | Out-File -FilePath "output.json" -Encoding utf8NoBOM

# 或者使用 .NET 类，更可控
[System.IO.File]::WriteAllText("output.json", $jsonString, [System.Text.UTF8Encoding]::new($false))
```

如果要兼容 Windows 记事本等上古程序，可以保留 BOM（`-Encoding utf8`），但现代工具链通常受不了 BOM，建议默认为无 BOM。

### 3. 管道传外部程序，先调好 `$OutputEncoding`

在 PowerShell 5.1 中，与外部程序交互前：

```powershell
$OutputEncoding = [System.Text.UTF8Encoding]::new($false)
```

这将管道输出编码改为 UTF-8 无 BOM。之后 `$jsonString | python script.py` 或者 `$jsonString | curl ...` 就能正确传递中文。

**注意：** 外部程序的 `$InputEncoding` 也可能影响，但多数情况只设置 `$OutputEncoding` 即可。若外部程序读取控制台输入，也需确保它是 UTF-8 输入模式。

### 4. 跨 PowerShell 版本的最佳实践：使用 Core（PowerShell 7+）

PowerShell Core（pwsh）将默认输出编码改为 UTF-8 无 BOM，`$OutputEncoding` 也默认为 UTF-8。这极大减少了“配置忘记”带来的坑。如果环境允许，应优先使用 PowerShell 7。Windows 自带的 PS 5.1 仅用于必须兼容的遗留系统，并在脚本开头显式设置编码。

## 踩坑点清单

- **IDE 终端编码混乱**：VSCode 终端可能配置为 UTF-8，而脚本执行环境继承系统代码页（如 936/GBK）。乱码时先检查 `[System.Console]::OutputEncoding` 和 `[System.Text.Encoding]::Default`，不一致时需调整。
- **BOM 残留导致 JSON 解析失败**：某些服务端用 UTF-8 带 BOM 的 JSON，`ConvertFrom-Json` 会直接抛错。解决方法：读取为字节流，跳过 BOM 三个字节后再解码。
- **Out-File 追加模式编码固定**：`Out-File -Append` 会沿用第一次写入时的编码，无法中途改变。避免混合编码的日志文件。
- **Invoke-RestMethod 的 `-ContentType` 参数只影响请求，不影响响应解析**。不要以为设置了请求 Content-Type 就能自动得到正确的响应编码。

## 可复用的模块化建议

将常用功能封装成函数，强制编码规范：

```powershell
function Invoke-API ($Uri) {
    $response = Invoke-WebRequest -Uri $Uri -UseBasicParsing
    $rawBytes = $response.RawContentStream.ToArray()
    $json = [Text.Encoding]::UTF8.GetString($rawBytes) | ConvertFrom-Json
    return $json
}

function Export-Text ($Path, $Content) {
    [IO.File]::WriteAllText($Path, $Content, [Text.UTF8Encoding]::new($false))
}
```

所有 API 脚本都通过统一入口执行，杜绝散落各处的 `>` 重定向。

## 总结

Windows 上 PowerShell 处理中文 JSON 的核心矛盾，在于 **PowerShell 的字符串内部是 Unicode，但文件的序列化、管道的传输边界却要靠工程师显式指定编码**。只要记住三条原则：**数据进出文件时显式指定 UTF-8 无 BOM**，**管道传输前调整 `$OutputEncoding`**，**对不确定来源的 JSON 先用字节流解码**，就能彻底告别中文乱码。

对于持续构建的 Agent 流水线，将编码策略固化为函数库，比事后盯着乱码日志找原因高效得多。编码问题从来不是技术难点，但它就像一个无声的测试：你的自动化链路，是不是每一段都说得清“字符集”这件事。

---

