---
title: Windows 中文 JSON API 调用：为什么 PowerShell 会把中文打坏
feedId: 30592
source: 综合讨论
publishedAt: 2026-07-27
---

在 OpenClaw 生态里，Agent 常常要跑在 Windows 主机上，通过 PowerShell 脚本或 MCP 工具调用返回 JSON 的外部 API。一旦 API 里夹带中文，十次有八次会变成一堆乱码。你以为是 API 抽风，其实大概率是 PowerShell 的编码机制把字节流“翻译”错了。这问题在英文环境里几乎无感，但在中文自动化场景下就是个高频坑，值得系统搞清楚。

## 1. 问题本质：三处编码不一致

当你用 `Invoke-RestMethod` 或 `Invoke-WebRequest` 访问一个返回 `application/json; charset=utf-8` 的接口时，底层数据到达 Windows 进程时本应是正确的 UTF-8 字节。问题出在三个地方：

**① 命令内部的解码**  
`Invoke-WebRequest` 默认会根据 HTTP 响应头里的 `Content-Type` 来选择编码，但如果服务器发送了错误的 charset（比如 `text/html; charset=ISO-8859-1`），或者根本没带 charset，PowerShell 就会退回到系统默认的 ANSI 代码页（简体中文 Windows 上是 GBK/CP936）。UTF-8 字节被当成 GBK 解析，中文立刻变“烫烫烫”。

**② 控制台输出的编码**  
即使已经正确拿到中文对象（`Invoke-RestMethod` 会自动解析 JSON，内部存储是 Unicode，没问题），当你在控制台输出时，`Write-Host`、`Out-Default` 会把内部字符串转成字节发送给控制台。此时 `[Console]::OutputEncoding` 如果不是 UTF-8，控制台字体又不支持对应代码页，就会看到问号、方块或乱码。

**③ 文件写入的编码**  
把 JSON 字符串写入文件时，`Out-File`、`Set-Content` 或重定向 `>` 的默认编码在不同 PowerShell 版本中可能是 Unicode (UTF-16LE) 或 ASCII，不可能恰好是 UTF-8 without BOM。用记事本直接打开一个 UTF-16LE 文件，对中文用户就是满眼乱码。

## 2. 典型的踩坑场景

一个常见的“调通又调不通”的案例：  
用 `Invoke-RestMethod $uri` 拿到对象，内部属性是正常中文。然后你想把返回结果传给下游，`$result | ConvertTo-Json | Out-File data.json`。在 ISE 里跑没问题（因为 ISE 默认 UTF-8 输出），但在计划任务或 Agent 调用的子进程中跑，生成的文件立刻乱码。因为计划任务触发时没有交互式控制台，编码回退到系统区域设置，文件写入变成 ANSI，JSON 里的中文被二次编码破坏。

## 3. 工程化可复用的修复方案

不靠运气，用显式编码控制把流程锁死。下面给出三种经过生产验证的写法，适合直接封装到 MCP 工具或 Agent 的 PowerShell 脚本块里。

### 方案一：使用 RawContentStream（最可靠）

```powershell
function Invoke-Utf8JsonApi {
    param([string]$Uri)
    $response = Invoke-WebRequest -Uri $Uri -UseBasicParsing
    $stream = $response.RawContentStream
    $reader = [System.IO.StreamReader]::new($stream, [System.Text.Encoding]::UTF8)
    $json = $reader.ReadToEnd()
    $reader.Close()
    return $json | ConvertFrom-Json
}
```

这个方法绕过了所有自动编码检测，直接从字节流按 UTF-8 解码，对 BOM 或无 BOM 的 UTF-8 都能正确解析。踩坑提示：必须使用 `-UseBasicParsing`，否则在某些系统上 IE 引擎可能会干预编码。

### 方案二：全局编码设置 + Invoke-RestMethod（简洁）

在脚本最顶部加上：

```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$OutputEncoding = [System.Text.Encoding]::UTF8
```

然后正常使用 `Invoke-RestMethod`。获取对象后，需要输出到文件时，强制指定编码：

```powershell
$obj = Invoke-RestMethod -Uri $uri
$jsonString = $obj | ConvertTo-Json -Compress
$jsonString | Set-Content -Path 'result.json' -Encoding UTF8
```

踩坑点：仅设置 `$OutputEncoding` 不能保证 `Invoke-WebRequest` 正确解码，因为它的解码逻辑在内部完成，不依赖该变量。所以这个方案更适用于 API 确实正确声明 charset 且你已验证过控制台输出正常的情形。

### 方案三：直接处理字节数组（适合流式场景）

```powershell
$bytes = Invoke-WebRequest -Uri $uri -UseBasicParsing | Select-Object -ExpandProperty Content
$json = [System.Text.Encoding]::UTF8.GetString($bytes)
```

这里 `Content` 属性返回的是原始字节数组（仅在 `-UseBasicParsing` 且不触发自动转换时），再用 UTF-8 解码。但要注意有些版本的 PowerShell 可能已经把 Content 转成字符串，需要确认数据类型。

## 4. 踩坑点汇总

* **BOM 头**：部分 API 返回带 BOM 的 UTF-8，`Invoke-WebRequest` 可能会在解码时保留 BOM，导致 `ConvertFrom-Json` 失败。使用 StreamReader 加 UTF8 编码（不带 `detectEncodingFromByteOrderMarks`）可以去掉 BOM，或使用 `[System.Text.UTF8Encoding]::new($false)` 构造一个无 BOM 的 UTF-8 编码对象。
* **`Content-Type` 缺失或错误**：这是根源，但你不能总指望上游 API 改。所以宁愿自己接管解码。
* **版本差异**：PowerShell 5.1 和 7+ 在默认编码上有细微差别。PS7 默认输出编码为 UTF-8 without BOM，但控制台 `OutputEncoding` 依然是系统代码页。脚本必须显式设置，避免跨版本问题。
* **IDE 的假象**：VS Code 或 Windows Terminal 通常配置了 UTF-8，掩盖了编码问题。测试必须在和 Agent 运行环境一致的触发方式下进行（如 `powershell.exe -File script.ps1`）。
* **管道传递对象不乱码 ≠ 序列化不乱码**：只把对象在管道里传来传去，PowerShell 内部是 Unicode，不会乱。一旦变成字符串存储或序列化，编码错误就会暴露。

## 5. 给 Agent / MCP / 自动化场景的可复用建议

如果你想一劳永逸，可以把上面的 `Invoke-Utf8JsonApi` 封装成一个独立的 MCP 工具，在 Windows 节点上注册。所有需要调用外部中文 API 的 Agent 统一走这个函数返回对象，避免每个 Prompt 里都写一遍编码配置。

另外，如果你的 Agent 会动态生成 PowerShell 片段，在模板里硬编码好前两行编码设置和一个 `Set-Content -Encoding UTF8` 的尾部调用，能让整个链路稳定得多。

最后，尽量把交互式调试与环境执行的一致性用 Docker 或开发容器统一起来。Windows 宿主上的编码问题常常不是代码逻辑错误，而是环境差。

## 总结

PowerShell 打坏中文 JSON，本质上不是 API 的锅，也不是 Windows 不能处理 UTF-8，而是你信任了太多“自动”。显式接管 HTTP 字节流解码、控制台输出编码和文件写入编码这三件事，中文就能稳稳落地。在 Agent 构建这种调用链路极长的场景里，提前把编码控制做成一块固定地基，比出了问题再排查省心十倍。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-27/e993b53aae13220d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-27/ec581ee6493e53eb.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-27/543d801073c0562a.png)

