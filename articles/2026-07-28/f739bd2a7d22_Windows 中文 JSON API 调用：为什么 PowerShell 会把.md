---
title: Windows 中文 JSON API 调用：为什么 PowerShell 会把中文打坏
feedId: 30709
source: 综合讨论
publishedAt: 2026-07-28
---

## 背景

在 Windows 上构建 Agent 或自动化流水线时，很多团队喜欢直接用 PowerShell 调用 REST API、解析 JSON 并传递中文内容。看似简单的操作，却经常莫名其妙地出现乱码：请求体的中文变成 `????`，或者响应里的中文字段直接被 `ConvertFrom-Json` 解析成乱码。尤其在 OpenClaw 这类需要拼接插件输出、MCP 消息传递中文给模型的场景，这种“无声损坏”会直接导致意图识别失败，排查起来又非常耗时。

这个问题并不是 PowerShell 的缺陷，而是 Windows 环境下 PowerShell 与 .NET 编码默认行为的叠加效应。下面从工程角度拆解根因，给出可复用的排查和修复方法。

## 问题根因：三条编码线不同步

PowerShell 处理字符串和外部系统交互时，实际上有三条独立的编码线在起作用：

1. **脚本文件的保存编码**  
   `.ps1` 文件本身用什么编码保存，决定了解析器看到的字符串原始字节。如果脚本中包含中文字面量，文件保存成 ANSI 或无 BOM 的 UTF-8，PowerShell 的解析器可能会按照系统区域设置（如 GBK）错误解读。

2. **进程的输出编码 `$OutputEncoding`**  
   控制 `>` 重定向、管道传给外部程序时使用的字节编码。默认为 ASCII（在 Windows PowerShell 5.1 中），这意味着你通过 `Invoke-RestMethod` 得到的中文，一旦通过管道传给 `Out-File` 或外部命令，会直接被阉割成 `?`。

3. **.NET 框架的 `[Console]::OutputEncoding` 与 BOM**  
   `Invoke-RestMethod` 内部调用 .NET 的 `HttpClient`，这些组件在序列化请求体时使用的是 `System.Text.Encoding.UTF8`（带 BOM？默认不带），但 `ConvertTo-Json` 在 PowerShell 5.1 中默认输出的字符串是 UTF-16 LE，导致后续写入或发送时发生字节混淆。

当这三条线不统一到 **UTF-8 无歧义的编码策略** 时，中文在任何一步都可能被损坏。

## 典型故障场景复现

假设我们需要调用某汉语分词 API，请求体如下：

```json
{"text": "今天天气真好"}
```

### 场景 1：直接用 `Invoke-RestMethod` 发中文

```powershell
$body = @{ text = "今天天气真好" } | ConvertTo-Json
Invoke-RestMethod -Uri $api -Method Post -Body $body -ContentType "application/json; charset=utf-8"
```

很多情况下 API 会收到 `????`，因为 `ConvertTo-Json` 产生的 `$body` 实际上是 .NET 字符串，其内部编码为 UTF-16，当 `-Body` 参数被转换成字节时，PowerShell 可能会使用 `$OutputEncoding`（ASCII）而非 UTF-8。即使 `-ContentType` 声明了 charset，实际的字节流已经损坏。

### 场景 2：保存响应到文件

```powershell
$resp = Invoke-RestMethod -Uri $api
$resp.result | Out-File output.txt
```

`Out-File` 默认使用 Unicode (UTF-16 LE)，但你可能期望 UTF-8。更糟糕的是，如果之前 `$OutputEncoding` 是 ASCII，则管道中间已经损坏，文件里只留下乱码。

### 场景 3：在 Agent 插件中拼接 JSON

插件从数据库读出中文，构建 JSON 模板，结果用 `Write-Output` 传出给宿主程序。宿主接收到的是被默认输出编码篡改后的字符串，MCP 消息发给模型时出现不可理解的内容。

## 做法/步骤：建立“全程 UTF-8”的通道

### 1. 保存脚本时强制 UTF-8 with BOM

在 VS Code 里面，底部状态栏点击编码，选择 `Save with Encoding` → `UTF-8 with BOM`。这保证脚本中的中文直接可见且被 PowerShell 正确解析。或者用命令批量转换：

```powershell
$content = Get-Content raw.ps1 -Raw
[System.IO.File]::WriteAllText("fixed.ps1", $content, [System.Text.UTF8Encoding]::new($true))
```

### 2. 全局设置输出编码

在脚本头部显式声明：

```powershell
$OutputEncoding = [System.Text.UTF8Encoding]::new($false)   # 无BOM
[Console]::OutputEncoding = [System.Text.UTF8Encoding]::new($false)
```

这个设置强制所有管道输出和外部程序通信使用 UTF-8。

### 3. 使用 .NET 方法序列化与发送

放弃 `ConvertTo-Json` 的隐式编码，改用 .NET 类库显式控制：

```powershell
$body = @{ text = "今天天气真好" }
$json = [System.Text.Json.JsonSerializer]::Serialize($body)   # 内部已为 UTF-8
$content = [System.Net.Http.StringContent]::new(
    $json,
    [System.Text.Encoding]::UTF8,
    "application/json"
)
$http = [System.Net.Http.HttpClient]::new()
$response = $http.PostAsync($api, $content).Result
$response.EnsureSuccessStatusCode()
$respBody = $response.Content.ReadAsStringAsync().Result
```

或者继续使用 `Invoke-RestMethod`，但手动将字符串转成 UTF-8 字节数组：

```powershell
$utf8 = [System.Text.Encoding]::UTF8
$bytes = $utf8.GetBytes($json)
Invoke-RestMethod -Uri $api -Method Post -Body $bytes -ContentType "application/json; charset=utf-8"
```

### 4. 写入文件时显式指定编码

```powershell
$utf8NoBom = [System.Text.UTF8Encoding]::new($false)
[System.IO.File]::WriteAllText("output.txt", $respBody, $utf8NoBom)
```

需要带 BOM 时用 `$true`。

## 踩坑点汇总

- **坑1：Windows PowerShell 的 `-Encoding` 参数陷阱**  
  `Out-File -Encoding UTF8` 这里的 `UTF8` 是 **带 BOM 的 UTF-8**（在 PowerShell 5.1），而某些 Linux 工具链（如 jq）可能因此解析异常。如果想用无 BOM，只能通过 .NET 实现。

- **坑2：字符串拼接的隐形破坏**  
  `"prefix $chineseString"` 本身不会破坏编码，但如果 `$chineseString` 来自被错误解码的外部源（如读文件用了 Get-Content 不指定 -Encoding UTF8），这个变量内部已经是垃圾。务必在获取数据时就明确编码。

- **坑3：`ConvertFrom-Json` 后的中文可能正常，但显示为乱码**  
  这通常是控制台字体或 `[Console]::OutputEncoding` 的问题，数据本身未损坏。可以用 `$str | Format-Hex` 检查字节，确认是 EF BB BF 开头的 UTF-8，或正确的 UTF-16 序列。

- **坑4：PowerShell 版本差异**  
  PowerShell Core（7.x）的默认 `$OutputEncoding` 已经是 UTF-8，问题主要集中在 Windows 自带的 5.1。如果生产环境仍在用 5.1，一定要自己打补丁到最新的 5.1 并配置编码策略。

## 可复用建议

- **建立一个 `Invoke-ApiUtf8` 模块**，封装上述编码控制逻辑，团队内所有 API 调用强制走这个函数。
- **在流水线脚本入口放置编码一致性检查**：  
  ```powershell
  if ($PSVersionTable.PSVersion.Major -lt 6 -and $OutputEncoding.IsSingleByte) {
      Write-Warning "OutputEncoding 不是 UTF-8，正在纠正..."
      $OutputEncoding = [System.Text.UTF8Encoding]::new($false)
  }
  ```
- **对于必须使用 ConvertTo-Json 的旧代码**，直接传入字节数组 `[System.Text.Encoding]::UTF8.GetBytes($json)` 而不是字符串，避免自动转换。
- **日志记录原始字节**，出现乱码时便于快速判定是写入环节还是显示环节损坏。

## 总结

PowerShell 在 Windows 下“打坏”中文，本质上不是脚本语言的问题，而是不同层级的编码默认值没有对齐。解决方案也很具体：从脚本保存、进程输出、序列化到文件写入，全链路显式指定 UTF-8，必要时用 .NET API 绕过 PowerShell 的隐式转换。对 OpenClaw 等需要可靠处理中文命令和模型输出的插件系统来说，这是一个投入小、收益大的稳定性加固点。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-28/3e73d3e11e47a817.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-28/4de16991d114febf.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-28/0b611d405aed3407.png)

