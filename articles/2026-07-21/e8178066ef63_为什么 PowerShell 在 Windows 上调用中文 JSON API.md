---
title: 为什么 PowerShell 在 Windows 上调用中文 JSON API 时总能打坏编码——一次彻底梳理
feedId: 29877
source: 综合讨论
publishedAt: 2026-07-21
---

## 背景

在 Windows 上跑自动化、Agent 或 MCP 插件时，经常会用 PowerShell 去调 HTTP API，通过 `Invoke-RestMethod` 或 `curl.exe` 发一段包含中文的 JSON 请求体。服务端拿到的却是 `???`、`锟斤拷` 或者赤裸裸的乱码，但同样的请求体在 Postman、Python 脚本里一切正常。这个问题在 OpenClaw 这类依赖 JSON 交互的工具链里尤其让人抓狂，因为一次编码错误就可能导致工具调用失败、workflow 中断。

很多人第一反应是“API 有问题”，但真正的问题是 Windows 上 PowerShell 的多个编码层一起作妖。

## 问题的根源：三重编码陷阱

### 1. 脚本文件本身的编码

你写了一个 `.ps1` 文件，里面硬编码了中文 JSON 字符串，或者在 IDE 里复制了一段 curl 命令。如果文件的保存编码是 `UTF-8 with BOM`，PowerShell 5.1 能正确解析；如果是纯 `UTF-8`（无 BOM），PowerShell 5.1 会按系统的 ANSI 代码页（如 GBK）去解读，中文当场变乱码。即使你在同一个文件里用到 `chcp 65001`，也无济于事，因为引擎解析脚本时就已经用错了编码。

### 2. `$OutputEncoding` 与重定向/管道

`$OutputEncoding` 这个变量控制 PowerShell 向外部进程发送数据时使用的字符编码。Windows PowerShell 5.1 中它的默认值是 `ASCII`（代码页 20127），即使你用 `echo '{"name":"你好"}' | curl.exe -d @- ...`，中文也会被强制转成 `?`。很多教程会让你设成 `[Console]::OutputEncoding`，但 Console 的输出编码经常是 `OEM` 代码页（如 437 或 936），仍然不是 UTF-8，问题继续存在。

### 3. `Invoke-RestMethod` 的 `-Body` 与 UTF-8

即便你避开了管道，直接在 `Invoke-RestMethod` 的 `-Body` 参数里传入一个对象或字符串，PowerShell 5.1 在序列化 JSON（`ConvertTo-Json`）和构造请求体时，仍然会根据当前会话编码，而不是硬编码 UTF-8。更隐蔽的是，即使你显式指定 `-ContentType "application/json; charset=utf-8"`，PowerShell 内部仍然可能先按 `ASCII` 或 `Unicode` 把字符串转成字节，导致服务端看到残缺的 UTF-8 序列。

## 可复现步骤

下面是用 Windows PowerShell 5.1 触发中文乱码的最小示例：

1. 确认 PowerShell 版本：`$PSVersionTable.PSVersion`，显示 `Major=5`。
2. 开启一个简单的 HTTP 回显服务（这里用 `httpbin.org`）：
   ```powershell
   Invoke-RestMethod -Uri "https://httpbin.org/post" -Method Post -Body '{"name":"测试"}'
   ```
   回显内容中 `data` 字段会看到 `name` 的值成了 `??` 或不可读字符。

3. 尝试显式指定 ContentType：
   ```powershell
   Invoke-RestMethod -Uri "https://httpbin.org/post" -Method Post -ContentType "application/json; charset=utf-8" -Body '{"name":"测试"}'
   ```
   在 5.1 中，问题依旧，因为字符串到字节的转换发生在 `WebRequest` 内部，它仍然依赖于 `[Text.Encoding]::Default`（多半是 GBK 或 ASCII）。

4. 用 `curl.exe` 和管道重试：
   ```powershell
   echo '{"name":"测试"}' | curl.exe -X POST -d @- "https://httpbin.org/post" -H "Content-Type: application/json"
   ```
   乱码，因为 `echo` 输出经过 `$OutputEncoding` 控制。

## 正确的工程化做法

### 方案一：全程强制 UTF-8（PowerShell 5.1 可用）

在脚本开头立即设置全局编码，并手动将请求体转为 UTF-8 字节数组：

```powershell
$PSDefaultParameterValues['*:Encoding'] = 'utf8'
[Console]::OutputEncoding = [Text.Encoding]::UTF8
$OutputEncoding = [Text.UTF8Encoding]::new()
$ProgressPreference = 'SilentlyContinue'

$body = @{ name = "测试" } | ConvertTo-Json
$bytes = [Text.Encoding]::UTF8.GetBytes($body)

Invoke-RestMethod -Uri "https://httpbin.org/post" -Method Post -Body $bytes -ContentType "application/json; charset=utf-8"
```

这里最关键的一点是**直接把字节数组传给 `-Body`**，此时 PowerShell 不会再用内部编码多转一次，而是原样发送。

### 方案二：迁移到 PowerShell 7

PowerShell 7（Core）在 Windows、Linux、macOS 上默认把 `$OutputEncoding` 设为 UTF-8，并且大部分 cmdlet 会在内部正确处理 UTF-8 编码。上面的第一个例子在 PS7 下直接运行就能得到正确结果。如果项目允许升级，这是最彻底的解法。

### 方案三：使用 `curl.exe`（并控制管道编码）

如果你必须用外部 curl，正确写法是：

```powershell
$body = '{"name":"测试"}'
$utf8Bytes = [Text.Encoding]::UTF8.GetBytes($body)
[Console]::OpenStandardInput().Write($utf8Bytes, 0, $utf8Bytes.Length) | curl.exe -X POST -d @- "https://httpbin.org/post" -H "Content-Type: application/json; charset=utf-8"
```

或者直接把请求体写入一个临时文件，并且用 `-d @tempfile.json` 指定文件，文件保存时使用 UTF-8 无 BOM 编码（可用 `Out-File -Encoding utf8NoBOM`）。

## 典型踩坑点

- **`ConvertTo-Json` 转义深度**：复杂对象转 JSON 深度不够时会丢失嵌套数据，建议加上 `-Depth 5` 或更高。
- **`-ContentType` 和 charset 冲突**：如果在 `-ContentType` 里写 `application/json; charset=utf-8`，PowerShell 5.1 可能把整个字符串当作 HTTP Content-Type 头直接发送，导致额外的 charset 被服务端误解析。更稳妥的做法是只用 `application/json`，然后单独通过 `-Headers` 或靠 `-Body` 字节数组保证编码。
- **ANSI 控制台的幽灵**：在命令行直接粘贴中文时，输入也可能被当前代码页（chcp）破坏。用脚本文件 + UTF-8 with BOM 规避。
- **`Set-Content` 等 cmdlet 默认编码**：它们默认是 `Default`（ANSI），生成临时文件时务必指定 `-Encoding UTF8`。

## 可复用建议

1. **编码自检函数**：在你的工具脚本开头跑一个自检，发包 `{"test":"你好"}` 到回显服务，检查返回的 `data` 是否包含正确中文，否则直接抛异常终止，避免后续流程静默乱码。
2. **环境统一化**：在自动化工作流（如 CI/CD 或 Agent 运行时）内使用 `$env:DOTNET_SYSTEM_GLOBALIZATION_INVARIANT=0`，避免某些 .NET 国际化回退导致的行为差异。
3. **抽象 HTTP 调用函数**：将上述字节数组 + Invoke-RestMethod 的逻辑封装成一个内部 `Invoke-Api` 函数，所有脚本统一调用，避免到处自行处理编码。
4. **优先使用 UTF-8 无 BOM 脚本**：但 PowerShell 5.1 脚本仍建议带 BOM，因为 ISE 和一些编辑器默认就是这样。使用 VSCode 时，设置 `"files.encoding": "utf8bom"` 专门给 `.ps1` 文件。

## 总结

Windows 上 PowerShell 调用中文 JSON API 时乱码不是“单个 Bug”，而是多个历史遗留的编码默认值叠加的结果：脚本文件编码、`$OutputEncoding`、`Invoke-RestMethod` 内部字符串到字节的转换策略。最可靠的工程解法是**显式使用 UTF-8 字节数组**，越过所有自动编码推断，让字节流直达 HTTP 请求体。如果你负责一套需要跑在 Windows 上的自动化工具链，花半小时把这个编码控制逻辑封装好，可以避免未来无数次毫无头绪的乱码排查。

记住：**在 PowerShell 里处理中文，永远不要相信任何默认编码。**

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-21/65217abdfb8aab1f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-21/699aedba8151d99c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-21/9dd253dafb750380.png)

