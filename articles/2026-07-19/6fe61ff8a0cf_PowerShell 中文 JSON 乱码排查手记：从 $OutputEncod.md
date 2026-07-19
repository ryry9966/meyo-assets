---
title: PowerShell 中文 JSON 乱码排查手记：从 $OutputEncoding 到 UTF-8 的工程实践
feedId: 29637
source: 综合讨论
publishedAt: 2026-07-19
---

## 背景：一次自动化脚本的“变音”事故

在 Windows 上用 PowerShell 调用第三方中文 API 时，你可能见过这样的返回：明明服务端日志记录的中文是“成功”，客户端收到的却是 `æˆåŠŸ`；或者你发送的 JSON 里中文变成了 `????`，接口直接报参数格式错误。这类问题在 Agent 自动编排、MCP 工具链以及各类无头 API 调用场景中尤其容易反复出现——因为脚本是静默运行的，没有肉眼确认的机会，失败往往只留下一堆难以追踪的乱码。

问题的根源并不复杂，但究其细节，恰恰是 Power­Shell 在 Windows 上的一套“历史兼容”编码行为，与当今 UTF-8 为王的 API 生态形成了系统性错配。本文将还原整个乱码链路，并给出确定性的排障与防御方案。

## 问题定位：UTF-8 下的三次重编码

PowerShell 调用 REST API，无外乎 `Invoke-RestMethod` 或 `Invoke-WebRequest`。当请求体（Body）包含中文时，乱码通常由以下三段中的至少一段诱发：

1. **字符串 → 字节流的转换**：PowerShell 把内存中的 .NET 字符串（UTF-16）转换为发送到网络的字节时，使用的编码取决于 `-ContentType` 参数、`$OutputEncoding` 变量以及命令自身的行为。  
2. **Content-Type 标注**：如果请求头中 `Content-Type` 没有带 `charset=utf-8`，某些 Web 框架会按 ISO-8859-1 解释字节，导致中文被破坏。  
3. **响应体的解码**：`Invoke-RestMethod` 在自动解析响应时，优先看响应头 `Content-Type` 的 charset；若缺失，在 Windows PowerShell 5.1 中可能回退到系统代码页（如 GBK/GB2312），而不是 UTF-8。

一个常见场景：你在 Windows PowerShell 5.1 中直接写：

```powershell
$body = @{ msg = "你好" } | ConvertTo-Json
Invoke-RestMethod -Uri https://api.example.com/echo -Method Post -Body $body -ContentType "application/json"
```

实际抓包会发现，发送的字节已经是 `ä½ å¥½` 之类的乱码，因为 `$body` 在转换为字节发送时使用了 `$OutputEncoding`（默认 ASCII），而非 UTF-8。

## 做法与步骤：从「写出事故」到「稳定交付」

下面给出一个在工程中可复制的最小稳健范式。假设目标环境是 **Windows PowerShell 5.1**（很多生产机至今未迁移），API 强制使用 UTF-8 编码的 JSON。

### 1. 确定性地生成 UTF-8 字节流

不要依赖隐式转换，直接使用 `[System.Text.Encoding]::UTF8.GetBytes()`：

```powershell
$payload = @{ action = "发送"; content = "中文内容" }
$jsonString = $payload | ConvertTo-Json -Compress
$jsonBytes = [System.Text.Encoding]::UTF8.GetBytes($jsonString)
```

### 2. 设置正确的 Content-Type 并发送

将字节数组直接作为请求体，并明确标注字符集：

```powershell
$response = Invoke-RestMethod -Uri "https://api.example.com/task" `
                              -Method Post `
                              -Body $jsonBytes `
                              -ContentType "application/json; charset=utf-8"
```

注意此时 **不要再** 将 `-Body` 绑定为字符串，否则 PowerShell 会用 `$OutputEncoding` 重新编码一次，抹掉我们刚做的 UTF-8 工作。

### 3. 修正响应解码环境（可选但推荐）

即使只是读取响应，Windows PowerShell 5.1 也可能误判编码。在脚本开始前可设置：

```powershell
[System.Net.ServicePointManager]::Expect100Continue = $false
$PSDefaultParameterValues['Invoke-RestMethod:ContentType'] = 'application/json; charset=utf-8'
$OutputEncoding = [System.Text.Encoding]::UTF8
```

`$OutputEncoding` 的修改会影响所有字符串到字节的转换行为，包括 `Invoke-WebRequest` 的字符串 Body 处理。如果你是面向大规模脚本集，建议统一到进程级别初始化。

### 4. PowerShell 7 的简便之道

如果不是必须拘泥于 Windows PowerShell，**直接使用 PowerShell 7** 会省去 80% 的烦恼。PowerShell 7 默认使用 UTF-8 无 BOM 作为 `$OutputEncoding`，且 `Invoke-RestMethod` 的字符处理更贴近 Web 标准。同等需求下只需：

```powershell
$body = @{ text = "中文" } | ConvertTo-Json
Invoke-RestMethod -Uri ... -Body $body -ContentType "application/json; charset=utf-8"
```

这是因为 `$OutputEncoding` 已经是 UTF-8，字符串到字节的转换不会再误用 ASCII。

## 踩坑点总览

- **依赖 Compress 后的汉字“原样”**：`ConvertTo-Json` 默认将非 ASCII 字符转义为 `\uXXXX`，这实际是安全的；但如果你先替换回汉字再发送，就又把编码风险引入了。
- **Content-Type 拼写不全**：写 `"application/json"` 而不加 `charset=utf-8`，部分服务端会进入猜测模式，多数 ISO-8859-1 猜测都会损坏中文。
- **$OutputEncoding 修改后忘记作用域**：在模块或脚本内修改该变量，可能污染其他命令，建议在脚本顶部分配并在结束时恢复，或仅通过显式字节流避免依赖。
- **将字节数组传递给 -Body 后仍妄图指定 -TransferEncoding**：`-Body` 为 byte[] 时，`Invoke-RestMethod` 会直接发送，除了 `Content-Type` 不应再追加其他编码头。
- **Windows PowerShell ISE 环境编码不同**：ISE 与普通控制台的 `$OutputEncoding` 可能不一致，排障时务必检查当前宿主（`$Host.Name`）。

## 可复用建议

1. **最低防御函数**：封装一个 `Invoke-ApiJsonUtf8`，内部强制使用 UTF-8 字节流，硬编码 `charset=utf-8`，让团队不再直接调用原生命令。
2. **环境自检脚本**：在 CI 开头加入 `$OutputEncoding.EncodingName -ne 'Unicode (UTF-8)'` 时的报警，避免在错误编码的代理机上运行。
3. **API 交互日志**：对发送 Body 和接收 Response 使用 `Out-File -FilePath ... -Encoding utf8` 记录原始字节的 Base64 形式，便于回溯时还原真实内容，而非凭控制台外观猜乱码。
4. **统一迁移到 PowerShell 7**：如果基础设施允许，在 Windows 上默认安装 PS7，将自动化入口指向 pwsh.exe 而不是 powershell.exe。

## 总结

PowerShell 把中文打坏，不是因为它不能处理中文，而是因为从 Windows 传统编码（系统 ANSI 页）到 UTF-8 的过渡期，留下了太多隐式编码假设。当你把字符串交给 `-Body` 时，其实已经隐式选择了某个编码器，而这个编码器在 Windows PowerShell 5.1 上默认是 ASCII，在 PS7 上是 UTF-8。忽略这个细节，就会在自动化链路的某个节点突然出现“随机”中文乱码。

解决方案的本质是**消除所有编码过程的模糊地带**：把字符串到字节的转换掌握在自己手中，明确告知服务端“我这是 UTF-8”，并确保响应解码的一致性。按此原则设计脚本，你的 Agent 再也不会因为一个“你好”而中断流程。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-19/829b6c8628a587b7.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-19/1c84b0c6352e7f29.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-19/3403ca2526692ca9.png)

