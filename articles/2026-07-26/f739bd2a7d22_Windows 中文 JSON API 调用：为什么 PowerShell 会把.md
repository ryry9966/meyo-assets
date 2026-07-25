---
title: Windows 中文 JSON API 调用：为什么 PowerShell 会把中文打坏
feedId: 30452
source: 综合讨论
publishedAt: 2026-07-26
---

## 背景

在 Windows 上构建 OpenClaw 插件、Agent 自动化或 MCP 服务时，我们经常需要用 PowerShell 调 REST API、解析 JSON 并把中文内容写进日志或传给下游。一个很典型的场景：用 `Invoke-RestMethod` 调某个 LLM Proxy，返回的 JSON 里 `content` 字段明明是一句通顺的中文，但在控制台打印出来却是 `æµ‹è¯•` 一类的乱码，或者直接导致 `ConvertFrom-Json` 报错。  

很多工程师第一反应是“API 返回的编码有问题”，但用 Postman 或 curl.exe 抓同一接口，中文完全正常。问题其实不在网络传输层，而在 **PowerShell 自身的编码决策链**。这篇帖子会拆解这个问题的根因，给出可复现的排查路径和工程化解决方案，避免各位在自动链路里反复掉坑。

## 问题定位：不是 JSON 坏了，是字符串被“翻译”错了

Windows 上的 PowerShell 有两个主流版本：  
- **Windows PowerShell 5.1**（基于 .NET Framework，系统预装）  
- **PowerShell 7+**（基于 .NET 6/8，跨平台，需单独安装）

5.1 版本为了兼容老旧控制台程序和 CMD，默认行为与系统的 ANSI 代码页（中文 Windows 上是 CP936，即 GBK）深度绑定。当 `Invoke-RestMethod` 或 `Invoke-WebRequest` 收到 HTTP 响应时，如果服务器返回的 `Content-Type` 没有明确标注 `charset=utf-8`，或者标注了但 PowerShell 认为“我需要把字符串交给控制台去显示”，它就会尝试用默认代码页去解码 UTF-8 字节流，结果自然就把多字节的中文字符打烂了。

更隐蔽的情况是：**即使你在控制台看到乱码，内存中的 `String` 对象可能是正确的**，但当这个字符串经过管道输出、被 `Out-File` 写入磁盘、或被其它进程通过标准输入读取时，又会经历一次编码转换。这就是为什么同样的 `Write-Host` 乱码，`curl | Out-File` 却不乱——因为不同输出目标的默认编码不同。

## 可复现步骤（基于 Windows 10/11 中文版）

1. **准备一个返回中文 JSON 的测试接口**  
   例如使用公共测试端点 `https://httpbin.org/anything`，用 `POST` 发送一个包含中文的 payload，它会原样返回。

2. **在 PowerShell 5.1 中执行如下脚本**：
   ```powershell
   $body = @{ text = "你好，世界" } | ConvertTo-Json
   $resp = Invoke-RestMethod -Uri "https://httpbin.org/anything" -Method Post -Body $body -ContentType "application/json"
   Write-Host $resp.json.text
   ```
   典型输出：`ä½ å¥½ï¼Œä¸–ç•Œ`。  
   如果改用 `Invoke-WebRequest` 并打印 `$resp.Content`，乱码可能不同，但都是非 UTF-8 解码的结果。

3. **对比 PowerShell 7 (pwsh.exe) 执行同一脚本**：输出正常中文。  

4. **进一步验证编码来源**：  
   在 PS 5.1 中执行 `[Console]::OutputEncoding`、`[System.Text.Encoding]::Default`，会发现它们指向 `System.Text.GB18030Encoding` 或类似的代码页；而在 PS7 中默认是 UTF-8 with BOM 或无 BOM 的 UTF-8。

## 根因解析

`Invoke-RestMethod` 内部流程大致是：  
1. 获取 HTTP 响应的字节流；  
2. 检查响应头 `Content-Type`，若无 `charset`，使用 **系统默认 ANSI 代码页** 解码；  
3. 返回解码后的字符串对象。  

对于绝大部分现代 API（特别是 OpenAI、Claude、阿里云、腾讯云这类服务），返回的 JSON 都是纯 UTF-8 编码，且在 `Content-Type: application/json` 中添加了 `charset=utf-8`。但不幸的是，PowerShell 5.1 在某些情况下会忽略这个 charset，或者虽然按 UTF-8 解码了字符串，但在 **输出到控制台显示** 时，`Write-Host` 又用 `[Console]::OutputEncoding` 将 UTF-16 内部字符串重新编码成字节，导致控制台再按 GBK 去解读，形成二次乱码。

此外，一个常见踩坑点是：**将 `$response` 直接通过管道写入文件**。  
`Out-File` 在 PowerShell 5.1 的默认编码是 `Unicode (UTF-16LE)`，而 `Set-Content` 默认跟随系统代码页。如果某个下游程序（比如 Python 脚本或日志分析器）期待 UTF-8，读到的却是 UTF-16LE 或 GBK 文件，又会继续产生乱码。

## 工程化解决方案

### 1. 首选：切换到 PowerShell 7
在 Windows 上安装 [PowerShell 7](https://aka.ms/powershell-release?tag=stable)，并在所有自动化脚本、OpenClaw 插件启动命令或 MCP 配置中显式使用 `pwsh.exe`。PS7 的 `$OutputEncoding` 默认为 UTF-8，且 `Invoke-RestMethod`、`Write-Host` 都遵循 UTF-8 逻辑，无需额外 hack。  
**对于 Agent 编排环境，这是成本最低的根治方案。**

### 2. 当必须使用 PowerShell 5.1 时的防御性编码
如果环境无法升级（例如某些企业级 Windows Server 锁死了版本），则在每个脚本开头强制设置输出编码：
```powershell
$OutputEncoding = [Console]::OutputEncoding = [Text.Encoding]::UTF8
```
这条语句同时覆盖了 `Out-File`/`Add-Content` 的默认重定向编码，以及控制台输出的显示编码。

此外，从 API 获取原始字节并手动解码，可以彻底绕过 cmdlet 内部的编码猜测：
```powershell
$resp = Invoke-WebRequest -Uri "https://api.example.com" -UseBasicParsing
$rawBytes = $resp.RawContentStream.ToArray()
$jsonString = [System.Text.Encoding]::UTF8.GetString($rawBytes)
$obj = $jsonString | ConvertFrom-Json
```
这种做法忽略 `Content-Type` 的 charset 猜测，直接以 UTF-8 读取，非常可靠。

### 3. 文件 I/O 的强制规范
- 写 JSON 日志、配置或中间结果时，永远加上 `-Encoding utf8NoBOM`：  
  `$json | Out-File -FilePath data.json -Encoding utf8NoBOM`  
- 如果用 `Set-Content`，同样要指定编码：`Set-Content -Path ... -Value $str -Encoding UTF8`。  
- 读入外部文件时，如果来源不确定，用 `Get-Content -Encoding UTF8` 避免默认代码页。

### 4. 在 OpenClaw/Agent 管道中的额外检查
当你的自动化脚本将 JSON 通过标准输出传给下一个 Agent 或 MCP 工具时，除了内容正确，还要确保输出流本身的编码不会被打断。一个简单验证：  
```powershell
$json = @{ content="中文测试" } | ConvertTo-Json
[Console]::OutputEncoding = [Text.Encoding]::UTF8
Write-Output $json
```
在外部（例如 Python 的 subprocess）用 `sys.stdout.buffer.read()` 检查接收到的字节是否为合法 UTF-8。如果仍然出现 GBK 编码，可能需要用 `[Console]::OpenStandardOutput().Write(...)` 直接操作字节流，但这类情况极少发生，多数是 `Write-Host` 和 `Write-Output` 混用导致的问题。

## 踩坑点汇总

- **PowerShell 5.1 默认编码不是 UTF-8**，且 `[Console]::OutputEncoding` 默认跟随系统区域设置。  
- **`Invoke-RestMethod` 对 charset 的处理并不总是遵守标准**：无 charset 时用系统编码，有 charset 时偶尔也会因内部实现差异而忽略。  
- **`Write-Host` 和 `Write-Output` 编码路径不同**：前者直接写到控制台主机，很可能用 `OutputEncoding`，后者通过管道输出编码可能被外部程序按当前代码页解释。  
- **`Out-File` 默认 UTF-16LE，`Set-Content` 默认系统代码页**，不要相信任何默认值。  
- **终端本身也可能有问题**：例如 Windows Terminal 里运行 PS5.1，字体缺字时显示为方块，这并非编码错误，但容易误导排查方向。

## 可复用建议

1. **统一 UTF-8 全线**：脚本开头设置 `$OutputEncoding`、`[Console]::OutputEncoding`，文件读写一律显式指定 `UTF8` 编码。  
2. **迁移到 PS7**：如果开发新插件或 Agent，直接要求 `pwsh` 作为运行时，并在文档中注明。  
3. **调试时用 `Format-Hex` 查原始字节**：当怀疑编码问题时，`$resp.RawContentStream | Format-Hex` 可以直接看到字节，摆脱字符显示层的误导。  
4. **为 API 封装统一函数**：例如 `Invoke-ApiUtf8`，内部用 `Invoke-WebRequest` + 手动 UTF-8 解码，避免每次重复编码陷阱。

## 总结

Windows 下 PowerShell 的中文 JSON 乱码问题，本质上是微软为兼容老旧控制台而留下的编码债务。对于现代 API、Agent 自动化、MCP 管线，UTF-8 早就是事实标准，而 PowerShell 5.1 的默认行为却还在使用系统代码页。一旦我们理解了“HTTP 字节流 → .NET String → 控制台/文件/管道”这个链条中每一次隐式编码转换，就能通过显式设置编码、升级运行时或操作原始字节流来根治问题。  

在 OpenClaw-CN 社区的实践中，这不仅仅是一个显示问题——它会导致 JSON 解析错误、下游处理的逻辑异常甚至 Agent 决策失败。因此，每次写自动化脚本前，都值得花两分钟设置好 `$OutputEncoding`，或者在工程规范里强制要求 PowerShell 7，让编码混乱不再成为阻塞项。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-26/32452dac33a35285.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-26/390009af36ee70ff.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-26/38625dcf1bd06c23.png)

