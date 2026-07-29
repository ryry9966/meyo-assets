---
title: Windows 中文 JSON API 调用必坑：为什么 PowerShell 会把中文打坏
feedId: 30915
source: 综合讨论
publishedAt: 2026-07-29
---

## 背景

在 Windows 上用 PowerShell 调用 JSON API 并处理中文内容时，你大概率会遇到这种场景：服务器返回的 JSON 明明一切正常，但你用 `Invoke-RestMethod` 或 `ConvertFrom-Json` 解析后，中文字符变成了一串问号、乱码甚至直接消失；更诡异的是，你把同样的 JSON 字符串粘贴到代码里测试，行为又完全正确。如果你是面向 OpenClaw、Agent、MCP 这类需要高度自动化、脚本化集成 RESTful API 的工程师，这种间歇性的编码问题会直接吃掉你两三个小时的排错时间。

这篇文章不讨论“字符集是什么”的基础概念，而是聚焦 Windows 上 PowerShell 处理文本管道时的真实行为，以及如何让你的自动化脚本在任何 Windows 机器上稳定地收发中文 JSON。

## 问题到底在哪里

表面现象是中文被打坏了，但根因通常不是 API 返回了错误编码，而是 **PowerShell 在将字节流转成字符串，或把字符串写入管道/文件时，使用了系统默认的 ANSI 代码页**。在简中 Windows 上，这个默认代码页通常是 936 (GBK)，而非 UTF-8。

具体会有三个高频触发点：

1. **Invoke-RestMethod 自动解析 JSON 时**  
   `Invoke-RestMethod` 会尝试自动将响应体反序列化为 PSCustomObject。如果服务器返回的 `Content-Type` 头里只写了 `application/json`，却没有明确 `charset=utf-8`，PowerShell 就可能按 ISO-8859-1 或系统代码页去解码字节流，然后才交给 JSON 解析器。此时高字节部分的中文已经损坏，后续再怎么转换也救不回来。

2. **将对象转为 JSON 字符串输出到文件或管道**  
   使用 `ConvertTo-Json` 生成了正确的 JSON 字符串，但当你通过 `Out-File` 或重定向符 `>` 将它写入文件时，PowerShell 再次自作主张地使用了 Unicode 重定向行为或 `Default` 编码。如果不显式指定 `-Encoding UTF8`，写入文件的字节序列并不是 UTF-8 编码，下游读文件的程序（比如 curl 或你自己的 HTTP body）就会看到乱码。

3. **`$PSDefaultParameterValues` 的隐式污染或控制台代码页设定**  
   某些自动化框架会改变 `[Console]::OutputEncoding`，或者你之前运行过一段修改代码页的脚本，导致整个会话的输出编码期望发生变化。这种改变对 `Invoke-WebRequest` 和 `Invoke-RestMethod` 的影响非常隐蔽，因为它们的编码回退逻辑依赖当前控制台的输出编码。

## 可复现的排障步骤

下面是一个最小可复现场景，你可以立刻在 Windows 10/11 的 PowerShell 5.1 或 PowerShell 7 上跑一遍，观察破坏过程。

**步骤 1：模拟一个返回中文 JSON 的 API。** 我们用 `netcat` 或 PowerShell 自建一个简单的 HTTP 监听器（省略搭建细节，假设 API 返回如下体）：
```json
{"message":"操作成功","code":200}
```

**步骤 2：PowerShell 5.1 的典型破坏请求：**
```powershell
$response = Invoke-RestMethod -Uri http://localhost:8080/api/demo -Method Get
$response.message   # 可能输出 "???" 或 "鎿嶄綔鎴愬姛"
```

**步骤 3：检查原始字节流。** 改用 `Invoke-WebRequest` 获取原始字节，然后手动按 UTF-8 解码：
```powershell
$raw = Invoke-WebRequest -Uri http://localhost:8080/api/demo -Method Get
$jsonString = [System.Text.Encoding]::UTF8.GetString($raw.Content)
$obj = $jsonString | ConvertFrom-Json
$obj.message  # 正常输出 "操作成功"
```

如果此时中文恢复，问题就锁定在 `Invoke-RestMethod` 的自动解码上。你可以通过抓到 `$raw.RawContent` 确认服务器返回的 `Content-Type` 中是否包含 `charset`。

**步骤 4：写入文件时的破坏。** 将上面正确的对象转为 JSON 再写入文件：
```powershell
$obj | ConvertTo-Json | Out-File -FilePath result.json
Get-Content result.json -Raw  # 中文正常
```
但在下次通过 curl 发送这个文件时，对方却收到乱码。用十六进制查看器打开 result.json，你会发现文件前两个字节不是 `EF BB BF`（UTF-8 BOM），而是直接以原始字节开头，且文件内中文用 ANSI 编码。此时使用 `-Encoding UTF8` 修复：
```powershell
$obj | ConvertTo-Json | Out-File -FilePath result.json -Encoding UTF8
```

## 踩坑点与工程化建议

- **不要相信 `Content-Type` 中会带 charset。** 即使在内部 API 中，开发者也很容易只写 `application/json`。在 PowerShell 5.1 里，最稳妥的办法是永远通过 `Invoke-WebRequest` 拿原始 byte[]，然后显式用 UTF8 解码，再交给 `ConvertFrom-Json`。为了代码可读性，你可以封装一个函数：
  ```powershell
  function Invoke-JsonApi {
      param($Uri, $Method = 'Get', $Body)
      $params = @{ Uri = $Uri; Method = $Method }
      if ($Body) { $params.Body = ($Body | ConvertTo-Json -Compress) }
      $response = Invoke-WebRequest @params
      $json = [Text.Encoding]::UTF8.GetString($response.Content)
      return $json | ConvertFrom-Json
  }
  ```

- **在脚本开头强制锁定输出编码。** 对于频繁输出 JSON 到文件或管道的脚本，显式设置 `$PSDefaultParameterValues` 会省去大量心思：
  ```powershell
  $PSDefaultParameterValues['Out-File:Encoding'] = 'utf8'
  $PSDefaultParameterValues['Set-Content:Encoding'] = 'utf8'
  ```
  注意，这不会影响重定向符 `>`。若要统一行为，可以在脚本开头将控制台输出编码也设为 UTF-8：
  ```powershell
  [Console]::OutputEncoding = [System.Text.Encoding]::UTF8
  ```

- **PowerShell 7 也有坑，但位置不同。** PowerShell 7 默认使用 UTF-8 而无 BOM，这修复了很多历史问题。然而当你在 Windows 上通过 `Start-Process` 或外部程序调用 pwsh 并依赖标准输出时，如果外部程序期望 ANSI 代码页，则可能反受其害。此时需要通过 `$env:PYTHONUTF8=1` 或设置 `$OutputEncoding` 来手动对齐。务必在自动化链路的上下游统一约定使用 UTF-8，并在交接点做显式编码声明。

- **JSON 对象到 curl 请求体时，绕过文件中间步骤。** 很多故障是因为先 `Out-File` 再 `curl -d @file` 造成的。你可以直接将 JSON 字符串通过管道传递给 curl 的 stdin：
  ```powershell
  $jsonStr = $obj | ConvertTo-Json -Compress
  $jsonStr | curl.exe -X POST -d @- http://api.local/endpoint -H "Content-Type: application/json; charset=utf-8"
  ```
  这样完全绕过了文件系统编码问题，且显式声明了 charset。

- **在 MCP/Agent 插件场景下的特殊注意点。** 如果你的 Agent 是通过 PowerShell 子进程调用本地工具，请确保启动子进程时指定了 `-NoProfile` 并且设置了 `$env:LC_ALL='C.UTF-8'`（或直接传递 UTF-8 编码的字节）。因为某些 Agent 框架会捕获标准输出并直接当作字符串解析，若 PowerShell 输出流编码与 Agent 预期不一致，中文就会在 IPC 边界被打坏。建议在插件内部使用 `[Console]::Error.Write` 输出调试信息，主数据通道只输出硬编码为 UTF-8 的 JSON 字符串。

## 总结

PowerShell 在 Windows 上处理中文 JSON 的“打坏”问题，本质上不是 JSON 解析器的缺陷，而是 PowerShell 与 Windows 历史遗留的 ANSI 默认编码之间的持续战争。对自动化工程师来说，掌握三个关键动作就可以彻底绕过：

1. 从 API 读取时，显式按 UTF-8 解码字节流再解析 JSON。
2. 将对象序列化为 JSON 后写入任何通道（文件、管道、进程间通信）时，强制指定 UTF-8 编码。
3. 在脚本顶部和子进程环境变量中，显式锁定整个管道的编码期望。

做到这三点，你就可以把“为什么 PowerShell 会把中文打坏”这个问题，从你的排障清单里永久划掉。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-29/42d3b80fec4ef199.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-29/9f7b197867093637.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-29/179e8bb22f8b4bfa.png)

