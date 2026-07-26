---
title: Windows 中文 JSON API 调用：为什么 PowerShell 会把中文打坏及工程化修复
feedId: 30505
source: 综合讨论
publishedAt: 2026-07-26
---

## 背景：来自 Agent 的一次乱码

当你在 Windows 上用 PowerShell 调用 OpenClaw 的 API 或者某个 MCP 工具，返回的 JSON 里包含中文昵称、Description 甚至 prompt 片段时，经常遇到一个尴尬场面：控制台输出变成 `éè¿åä½` 这类不可读字符，或者写入文件后解析器报 `Invalid UTF-8`。Agent 流程因此中断，而排查时又容易在 IDE、终端、重定向之间反复横跳。

这个问题在自动化管道里会被放大——一旦某个中间脚本把 JSON 打坏，下游的 jq 解析、插件配置生成、MCP 工具输入都会连锁失败。本文不会搬运泛化鸡汤，只给出能在你的工程里立刻复用的成因分析和修复方案。

## 问题显微：编码管道里谁在“转码”

PowerShell 5.1（Windows 自带）和 PowerShell 7+ 在编码行为上有微妙差异，但根源一致：

1. **API 响应的原始字节**：绝大多数现代 API 返回的 JSON 使用 UTF-8，没有 BOM。
2. **Invoke-RestMethod / Invoke-WebRequest 解析**：这两个 cmdlet 会尝试把响应用 .NET 字符串表示。.NET 内部是 UTF-16，转换时必须知道原始编码。如果服务端 Content-Type 头缺失 charset，或者声明了错误的编码，.NET 可能按 ISO-8859-1 或系统默认 ANSI 代码页（通常是 GBK/936）去解码，导致非 ASCII 字符损坏。
3. **从进程缓冲区到控制台**：即使 .NET 内部字符串正确，PowerShell 输出到控制台或文件时，还会经过 `[Console]::OutputEncoding` 的再编码。Windows 控制台默认代码页 936，它会试图把正确字符串映射到窄字符编码，造成问号或乱码。
4. **重定向与管道**：`>` 和 `Out-File` 默认使用 `Unicode`（UTF-16 LE）或跟随 `$OutputEncoding`，而很多自动化工具期望 `UTF-8 no BOM`。一旦带上 BOM，后续 `ConvertFrom-Json` 可能直接抛异常。

举一个可直接复现的问题场景：

```powershell
$resp = Invoke-RestMethod -Uri "https://api.example.com/item" 
$resp.title  # 控制台显示乱码
$resp | ConvertTo-Json > result.json  # 文件用记事本打开正常，但程序读报错 BOM 或 UTF-16
```

## 根因定位与复现办法

在工程机器上执行以下步骤，可以清晰看到编码链的断裂：

```powershell
# 1. 查看当前控制台编码
[Console]::OutputEncoding

# 典型输出：
# IsSingleByte      : True
# BodyName          : gb2312
# EncodingName      : 简体中文(GB2312)

# 2. 查看 PowerShell 输出编码（会影响到 Out-File 和重定向）
$OutputEncoding

# 5.1 下通常是：ASCII（真坑）; 7+ 下通常是 UTF-8
```

然后调用一个返回中文 JSON 的测试端点：

```powershell
$json = Invoke-RestMethod -Uri "https://httpbin.org/anything" -Method GET -Headers @{accept='application/json'}
# httpbin 会回显你的请求，如果有中文参数可以看到乱码效应
$json | Out-File -FilePath test.json
Get-Content test.json
```

此时大概率出现中文损坏。若在 VS Code 终端（强制执行 UTF-8）运行，症状可能消失，这恰恰说明问题来自系统级编码配置。

## 工程化修复：一次性对齐所有环节

### 方案一：控制台与变量编码强设为 UTF-8（推荐，适用于长期项目）

在你的 PowerShell 配置文件 `$PROFILE` 中添加：

```powershell
# 强制控制台输出使用 UTF-8
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
# 将管道与文件重定向输出编码也设为 UTF-8（PS 5.1 需要）
$OutputEncoding = [System.Text.Encoding]::UTF8
# 对于接收外部命令输出，设置输入编码
[Console]::InputEncoding = [System.Text.Encoding]::UTF8
```

然后 **重启 PowerShell**。此后 `Out-File` 不带 `-Encoding` 时就会输出 UTF-8 无 BOM（PS 5.1 仍需显式指定，见下文）。

**Note**：在 PowerShell 5.1 里，`$OutputEncoding` 只影响管道到外部命令的编码转换，并不能彻底改变 `>` 重定向运算符的行为。所以更稳妥的办法是永远显式指定编码。

### 方案二：在每次 IO 操作中显式声明编码

```powershell
# 获取响应，直接以 Raw Byte 形式处理，避免 .NET 自动解码错误
$raw = Invoke-WebRequest -Uri "https://api.example.com/item" -UseBasicParsing
$utf8 = [System.Text.Encoding]::UTF8
$jsonStr = $utf8.GetString($raw.RawContentStream.ToArray())
# 现在 jsonStr 是正确字符串

# 写入 UTF-8 without BOM
[System.IO.File]::WriteAllText("$PWD/result.json", $jsonStr, [System.Text.UTF8Encoding]::new($false))
```

如果你仍惯用 `Invoke-RestMethod`，可加上自动应答头来暗示编码：

```powershell
$headers = @{ 'Accept-Charset' = 'utf-8' }
$resp = Invoke-RestMethod -Uri "..." -Headers $headers
```

但此法不总是可靠，客户端强制指定字符集优先级低于服务端声明。

### 方案三：绕过 PowerShell 的默认编码——用 `curl.exe`

PS 的 `curl` 是 `Invoke-WebRequest` 的别名，在需要确定行为时，直接调用系统自带的 curl.exe：

```powershell
curl.exe -s https://api.example.com/item | Out-File -Encoding UTF8 item.json
Get-Content item.json -Encoding UTF8 | ConvertFrom-Json
```

特别注意 curl.exe 的 `-s` 避免进度干扰，`-o` 直接写文件效果更好：

```powershell
curl.exe -s -o item.json https://api.example.com/item
```

文件是原始的字节流，保留服务器发送的 UTF-8 编码，没有二次编码风险。

## 踩坑手记：BOM 隐雷与 jq 崩溃

- **BOM 问题**：`Out-File -Encoding UTF8` 会输出带 BOM 的 UTF-8。很多 JSON 解析器（包括 jq、PowerShell 的 `ConvertFrom-Json`）遇到 BOM 头会罢工。正确写法：
  ```powershell
  $utf8NoBOM = New-Object System.Text.UTF8Encoding $false
  [System.IO.File]::WriteAllText("$PWD/clean.json", $content, $utf8NoBOM)
  ```
  或 PS 7+：
  ```powershell
  $content | Out-File -Encoding utf8NoBOM clean.json
  ```

- **Invoke-WebRequest 的 `Content` 属性**：已经过内部编码转换，可能已经损坏。建议直接操作 `RawContentStream` 或使用 `Invoke-RestMethod` 返回的 PSCustomObject，但只在确认编码正确时才可靠。

- **cmd.exe 内嵌调用**：如果你的 Agent 流程通过 cmd 启动 powershell.exe 并有输入重定向，编码完全是另一个噩梦。优先全部切换到 pwsh 7+，并统一使用 UTF-8 参数。

## 可复用建议清单

1. **标准化执行环境**：在所有开发机、CI 代理上安装 PowerShell 7+，并在脚本首行设置 `[Console]::OutputEncoding = [Text.Encoding]::UTF8`。
2. **写文件永远显式编码**：弃用 `>` 和 `Out-File` 无参数形式，强制用 `Set-Content -Encoding UTF8NoBOM` 或 .NET 的 `WriteAllText`。
3. **API 数据传输用二进制思考**：当编码混乱时直接读取字节流解码，不要信任中间层。
4. **Agent 配置防御**：为 MCP 或插件脚本增加编码自检，比如检验输出 JSON 能否被 `Test-Json` 通过，若不通过则回退到重试或修复。
5. **统一 BOM 策略**：团队明确 JSON 使用 UTF-8 without BOM 作为唯一标准，并在代码仓库添加 `.gitattributes` 标注文本文件编码。

## 总结

PowerShell 把中文 JSON 打坏的根因是控制台默认代码页、输出编码与 UTF-8 无 BOM 之间的多层阻抗不匹配。在自动化流程里这个问题被放大，但修复思路很直接：把整个数据处理链显式锁定在 UTF-8（无 BOM）上，无论控制台还是文件，不留隐式转换空间。几个关键配置加上文件写入的显式编码习惯，就能让你的 OpenClaw 插件、MCP 工具和 Agent 管道告别天书乱码。

后续如果你的 Windows Agent 依然遇到诡异字符问题，不妨先跑一遍 `chcp 65001` 并检查 `$OutputEncoding`，往往能省下大量排障时间。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-26/e645d4550f51fa6f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-26/c2bf6b887cc5ae63.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-26/93de560e10926e8b.png)

