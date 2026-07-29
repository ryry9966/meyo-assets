---
title: Windows 中文 JSON API 调用：为什么 PowerShell 会把中文打坏，以及如何根治
feedId: 30899
source: 综合讨论
publishedAt: 2026-07-29
---

## 背景

很多 OpenClaw 的 MCP 工具、Agent 插件或自动化实践，都绕不开一个基础动作：在 Windows 上用 PowerShell 调用 APIs 拿到中文内容，再传给下游处理。比如用 `Invoke-RestMethod` 拉取知识库摘要、从翻译接口获取结果、或为某个 MCP 服务器实现一个“查询天气/新闻”的工具。

在本地开发时脚本跑得没问题，但到了生产环境、计划任务或作为子进程被 OpenClaw 调用时，中文经常变成一堆 `?`、`□□` 甚至不可逆的乱码。更糟的是，你以为修好了，换台机器又坏掉——根源在于 PowerShell 的编码行为远比看起来复杂。

本文把问题拆开，给出一套可复现、可落地的根治方案，并提炼出几个可以直接搬进 `README` 的工程建议。

## 问题复现

用一个最小案例就能触发。假设我们请求一个返回中文 JSON 的测试接口（比如 `https://httpbin.org/anything?msg=你好`），在 PowerShell 5.1 里执行：

```powershell
$resp = Invoke-RestMethod -Uri 'https://httpbin.org/anything?msg=你好'
$resp.args.msg  # 预期输出：你好
```

在你的开发机上可能正常，但在另一台 Windows Server 2016 上，控制台会打出 `??`。如果把这个变量写入文件再交给 OpenClaw 解析：

```powershell
$resp | ConvertTo-Json | Out-File result.json
```

打开 `result.json` 发现中文全部损坏，下游工具无法解析，整个链路中断。

## 根因分析

PowerShell 5.1 的编码行为有四个常见陷阱，互相叠加：

1. **Web cmdlet 的响应解码不遵守 UTF-8**  
   `Invoke-RestMethod` 和 `Invoke-WebRequest` 默认依据 HTTP 响应头中的 `charset` 解码。如果服务端没有明确返回 `charset=utf-8`，PowerShell 会退回到 **ISO-8859-1**（Windows-1252 的近似），于是中文字节被错误映射，造成不可逆损毁。

2. **输出编码（`$OutputEncoding`）默认不是 UTF-8**  
   `$OutputEncoding` 控制着数据重定向到外部进程时的编码。默认为 ASCII（或系统 OEM 代码页），导致传给 OpenClaw 的 stdout 也是坏的。

3. **控制台编码（`[Console]::OutputEncoding`）不匹配**  
   在 Windows 控制台，很多字体不支持中文显示，但更本质的是，如果控制台编码不是 UTF-8，即使数据本身正确也可能显示为乱码。但这属于显示问题，数据损坏通常发生在步骤 1 和 2。

4. **重定向与文件写入默认使用 ANSI**  
   `Out-File` 和 `>` 运算符在没有 `-Encoding` 参数时，默认编码是 **Unicode (UTF-16LE)** 在 PowerShell 5.1 中，但实际写入文件的行为依赖于 `$PSDefaultParameterValues` 和重定向实现，容易出现不确定性。更常见的是使用 `Set-Content` 时默认 ASCII，导致文件坏掉。

综合结果：API 返回的 UTF-8 字节流被错误解码，修不好；或者数据本身正确，但输出给下一个进程（比如 OpenClaw 的 MCP 服务器进程）时再次被转码打坏。

## 根治步骤

### 1. 强制 UTF-8 解码 API 响应

如果 API 的 `Content-Type` 头里不含 `charset=utf-8`，需要显式处理。两种方案：

**方案 A：用 `-ContentType` 发起请求，并接收原始字节手动解码**

```powershell
$response = Invoke-WebRequest -Uri $uri -ContentType "application/json; charset=utf-8"
$rawBytes = $response.Content -as [byte[]]  # 注意：Content 是字符串，需转为字节
$utf8String = [System.Text.Encoding]::UTF8.GetString($rawBytes)
$obj = $utf8String | ConvertFrom-Json
```

不过 `Invoke-WebRequest` 的 `.Content` 属性已经是根据响应头解码后的字符串，此时若响应头缺失 charset，数据已损坏。真正可靠的做法是直接从 `RawContentStream` 读取：

```powershell
$resp = Invoke-WebRequest -Uri $uri -UseBasicParsing
$stream = $resp.RawContentStream
$reader = [System.IO.StreamReader]::new($stream, [System.Text.Encoding]::UTF8)
$json = $reader.ReadToEnd()
$obj = $json | ConvertFrom-Json
```

**方案 B（推荐）：切换到 PowerShell 7**  
PowerShell 7+ 的 `Invoke-RestMethod` 已改为默认假设响应为 UTF-8，当 charset 缺失时不再退回到 ISO-8859-1，大幅减少此类问题。

### 2. 统一全局 UTF-8 编码

在脚本开头或 profile 中设定：

```powershell
$OutputEncoding = [System.Text.Encoding]::UTF8
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
```

对于 PowerShell 5.1 文件写入，务必显式指定 `-Encoding UTF8`：

```powershell
$obj | ConvertTo-Json | Out-File -Encoding utf8 result.json
```

或改用 `Set-Content -Encoding utf8`。

### 3. 作为 MCP 工具输出时防止二次编码损坏

当你的脚本被 OpenClaw 作为子进程调用，stdout 是传递数据的唯一通道。确保：

- PowerShell 脚本输出使用 `Write-Output`，不要使用 `Write-Host`（后者不进管道）。
- 主脚本开头强制设置：
  ```powershell
  [Console]::OutputEncoding = [System.Text.Encoding]::UTF8
  ```
- 调用脚本的外部进程应设置环境变量 `PYTHONIOENCODING=utf-8` 或相应语言配置，接收方也需以 UTF-8 读取子进程输出。

### 4. 验证

在脚本末尾加入一个自检，免得半吊子修复：

```powershell
$test = [System.Text.Encoding]::UTF8.GetString([System.Text.Encoding]::UTF8.GetBytes('你好'))
if ($test -ne '你好') { throw 'UTF-8 roundtrip failed' }
```

## 踩坑点实录

- **`Invoke-RestMethod` 返回的是 PSCustomObject 而非原始 JSON**  
  如果下游必须拿到原始 JSON，直接用 `Invoke-WebRequest` 取 `.Content`，并遵循上述手动解码流程。`Invoke-RestMethod` 的自动解析已经损坏了数据，无法通过重新编码恢复。

- **PowerShell ISE 与 VSCode 编码表现不同**  
  别太相信 ISE 的输出，它有时能正确显示只是因为它内部用了不同的显示编码，掩盖了数据已经损坏的事实。一定要写到文件里再检查 hex 值。

- **BOM 问题**  
  `Out-File -Encoding utf8` 产生的文件带 BOM，某些 JSON 解析器会报错。使用 `-Encoding utf8NoBOM`（PowerShell 7 支持）或用 `.NET` 的 `[System.IO.File]::WriteAllText()` 以 UTF-8 无 BOM 写入。

- **系统代码页变更不生效**  
  有些教程让运行 `chcp 65001`，这只影响当前控制台会话，且不改变 `[Console]::OutputEncoding` 内部值，所以不要依赖它。

## 可复用建议

如果你正在构建会被分发到不同 Windows 节点的自动化工具，可以把下面这些写进代码规范或 README：

1. **脚本顶部模板**（适用于 PowerShell 5.1 兼容）：
   ```powershell
   [Console]::OutputEncoding = [System.Text.Encoding]::UTF8
   $OutputEncoding = [System.Text.Encoding]::UTF8
   $PSDefaultParameterValues['Out-File:Encoding'] = 'utf8'
   $PSDefaultParameterValues['*:Encoding'] = 'utf8'
   ```
   并手动处理 API 响应的 UTF-8 解码。

2. **优先使用 PowerShell 7**，在 Docker 容器中或通过 `pwsh` 调度。这是最低摩擦的道路。

3. **永远不要假设控制台输出代表数据真实内容**，校验文件或管道下游。

4. **MCP 工具设计时**，让工具脚本输出统一为 `ConvertTo-Json` 并 `Write-Output`，并在调用端以 UTF-8 解码读取子进程 stdout。

## 总结

Windows 上 PowerShell 的中文乱码，本质是多层编码不一致的叠加：HTTP 响应解码、进程内字符串编码、输出编码、文件写入编码。只修一端往往无效，需要一套组合拳：强制 UTF-8 解码源头数据、统一进程内输出编码、显式控制文件写入。一旦把这几个点固化进工具模版，OpenClaw 的 MCP 工具就再也不会因为“中文坏了”而静默失败。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-29/ef83d5fb3eb284b7.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-29/881ed4b7d603f550.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-29/3130ec08ce1183a9.png)

