---
title: Windows 下调用 JSON API 中文乱码？带你从 PowerShell 编码坑里爬出来
feedId: 30408
source: 综合讨论
publishedAt: 2026-07-25
---

在 OpenClaw、插件、Agent 或 MCP 的自动化实践中，Windows 上使用 PowerShell 调用 JSON API 是再常见不过的场景。无论是向本地模型服务发送 prompt，还是读取配置、构建消息体，一旦涉及中文，你很可能遇到过“好好的中文被 PowerShell 打坏了”的诡异现象——服务端收到一堆 `????`、`锟斤拷` 或者字符串里凭空多出转义。这篇文章不兜圈子，直接定位根因，给出可复现的修复方案和工程化建议。

---

## 一、为什么 PowerShell 这么容易把中文搞坏？

关键在于**编码的不一致**。Windows 系统内部大量使用 UTF-16 LE，而网络世界以 UTF-8 为主导。PowerShell 在两个世界之间转换时，只要有一个环节的编码假设出错，中文就会在字节流转中“烂掉”。常见触发点有四个：

1. **脚本文件的保存编码**：在 ISE 或记事本中写的脚本，默认可能是 ANSI 或 UTF-8 with BOM，而其中定义的中文字符串常量，在运行时可能被错误解释。
2. **控制台输出/输入编码**：`[Console]::OutputEncoding` 决定了控制台显示的编码，但它也影响管道和重定向时的字节流编码。很多人在交互窗口看到乱码，就以为只是显示问题，其实它会影响 `Invoke-RestMethod` 的 body 构造。
3. **文件读写编码**：`Get-Content`、`Out-File`、`Set-Content` 在不同 PowerShell 版本中默认编码完全不同——Windows PowerShell 5.1 的 `> ` 输出重定向会生成 UTF-16 LE 文件；而 `Get-Content` 默认探测编码，结果往往不是 UTF-8。
4. **HTTP 请求体与响应的编码协商**：`Invoke-RestMethod` 在构造请求时，即使指定了 `-ContentType "application/json; charset=utf-8"`，如果你传入的 `-Body` 是已经“坏掉”的字符串，服务端收到的就是乱码字节。

这些机制纠缠在一起，导致同一个 API 用 `curl` 能正常调用，换成 Power​​Shell 就面目全非，而新手往往困在表面现象上反复试错。

---

## 二、从问题到可复现最小例子

假设你有一个本地 API 接收 JSON 并原样返回，用来调试。下面这个看似无害的 PowerShell 片段就会触雷：

```powershell
$body = @{ message = "你好，世界" } | ConvertTo-Json
Invoke-RestMethod -Uri http://127.0.0.1:8000/echo -Method Post -Body $body -ContentType "application/json; charset=utf-8"
```

在 Windows PowerShell 5.1 下，`ConvertTo-Json` 默认将字符串转义为 `\uXXXX` 形式？不，问题不在转义，而在 `$body` 所对应的 .NET 字符串是 UTF-16，但 `Invoke-RestMethod` 在序列化成字节流发送时，会依赖 `[Text.Encoding]::Default` 或者会话的编码设置。如果你没有提前干预，它可能会使用系统的 ANSI 代码页（如 GBK 936），这样“你好”就变成 GBK 字节，而服务端如果按 UTF-8 解码，自然出现乱码。

另一个典型场景：API 返回中文，但 PowerShell 控制台打印出乱码。例如：

```json
{ "reply": "操作成功" }
```

你用 `$res = Invoke-RestMethod ...` 然后 `$res.reply` 得到的是乱码。这是因为响应体被当作 ISO-8859-1 或系统 OEM 代码页解码了，而实际内容却是 UTF-8。

---

## 三、可靠做法：从源头控制编码

### 1. 在脚本顶部固定编码环境

无论脚本在哪个 Windows 主机上运行，首先强制设定成 UTF-8：

```powershell
$OutputEncoding = [Console]::InputEncoding = [Console]::OutputEncoding = [System.Text.Encoding]::UTF8
```

这三行保证：控制台输入输出、以及与外部进程（如 python、curl）交互时的管道编码全部统一为 UTF-8。注意 `$OutputEncoding` 是 PowerShell 特有的变量，影响 `>` 和 `|` 等操作符生成输出的编码。

此外，如果控制台活动代码页不是 65001，在执行某些 exe 时仍可能出问题，可以顺带执行：

```powershell
chcp 65001 > $null
```

但仅 `chcp` 不足以修正 PowerShell 内部的字符串处理，所以上面的变量设置才是关键。

### 2. 构造 JSON 请求体时避免手动拼接字符串

推荐让 PowerShell 自动处理序列化：

```powershell
$payload = @{
    message = "你好，世界"
    timestamp = Get-Date -Format o
}
$response = Invoke-RestMethod -Uri $uri -Method Post -Body ($payload | ConvertTo-Json -Depth 5) -ContentType "application/json; charset=utf-8"
```

若要更精确地控制 JSON 输出（比如避免 `ConvertTo-Json` 对特殊字符的转义），可以改用 `[System.Text.Json.JsonSerializer]::Serialize`，但通常上面的方式已经足够。关键是 **`-ContentType` 必须显式包含 `charset=utf-8`**，否则某些服务器或 .NET 实现可能回退到错误编码。

如果你确实需要自己生成 JSON 字符串，那要把字节序列化包起来：

```powershell
$jsonString = '{"message":"你好"}'
$bytes = [System.Text.Encoding]::UTF8.GetBytes($jsonString)
Invoke-RestMethod -Uri $uri -Method Post -Body $bytes -ContentType "application/json; charset=utf-8"
```

### 3. 读写文件时显式指定 UTF-8

- 读取 JSON 配置文件：
  ```powershell
  $config = Get-Content -Path config.json -Encoding UTF8 -Raw | ConvertFrom-Json
  ```
- 保存结果：
  ```powershell
  $response | ConvertTo-Json -Depth 5 | Out-File -FilePath result.json -Encoding utf8
  ```
- 如果是写日志，使用 `Add-Content -Path log.txt -Value $line -Encoding utf8`。

在 Windows PowerShell 5.1 中，一定要注意 `>` 重定向符会产生 UTF-16 LE 文件，不要用它保存含中文的数据；改用 `Out-File -Encoding utf8` 或 `Set-Content -Encoding utf8`。

### 4. 处理响应乱码

如果 API 返回的 Content-Type 里没有 `charset=utf-8`，`Invoke-RestMethod` 可能按默认 ISO-8859-1 解码。你可以用 `Invoke-WebRequest` 获取原始字节自行解码：

```powershell
$response = Invoke-WebRequest -Uri $uri -Method Post -Body $body -ContentType "application/json; charset=utf-8"
$rawBytes = $response.RawContentStream.ToArray()
$decoded = [System.Text.Encoding]::UTF8.GetString($rawBytes)
```

或者更简单：如果服务端总是 UTF-8，但忘记声明，可以在请求时加上 `-Headers @{ "Accept-Charset" = "utf-8" }`，但这取决于服务端是否尊重。

---

## 四、踩坑点与排障方法

以下是实际踩过的坑，以及对应的快速检查方法：

- **坑 1**：在 ISE 中运行脚本正常，但从 `.ps1` 文件调用时乱码。  
  *原因*：文件保存编码是 ANSI，但内部字符串在解析时可能被误读。  
  *解决*：将脚本文件另存为 **UTF-8 with BOM**（Windows PowerShell 5.1 需要 BOM 才能正确识别 UTF-8），或者迁移到 PowerShell 7（默认 UTF-8）。

- **坑 2**：`ConvertTo-Json` 导致中文变成 `\uXXXX`。  
  *现象*：JSON 里中文全变成转义序列，但对于 API 调用，这其实不是乱码，而是合法的 JSON。如果服务端不能处理，就需要使用其他序列化器。通常大部分现代 API 都能正确解析，不必特别解决。若非要中文原文，可以在生成 JSON 后：

  ```powershell
  [Regex]::Unescape($json)
  ```

  但要注意这可能破坏其他转义，慎用。

- **坑 3**：在管道中混合了外部命令和 Power​​Shell cmdlet，编码突变。  
  例如将 `curl.exe` 的输出通过 `|` 传到 `ConvertFrom-Json`，如果 `$OutputEncoding` 未设成 UTF-8，数据就被破坏了。确认所有外部命令的输入/输出与 PowerShell 之间的管道编码匹配。

**快速排障清单**：
1. 在脚本最前面输出当前编码：
   ```powershell
   [Console]::OutputEncoding.EncodingName    # 期望：Unicode (UTF-8)
   $OutputEncoding.EncodingName
   ```
2. 用 `curl.exe` 或 Postman 先行验证 API 本身的正确性。
3. 在 `Invoke-RestMethod` 前，用 `Write-Host $body` 检查即将发送的字符串是否正常（需设置终端为 UTF-8 才能正确显示）。
4. 如果响应乱码，先打印 HTTP 响应头中的 `Content-Type`：`$res.Headers["Content-Type"]`，确认服务端声明了 charset。

---

## 五、可复用的自动化建议

在 Agent、MCP 这类需要频繁调用 API 的工程里，建议将这些编码规则固化为标准片段：

- 创建一个 `Init-PowerShellUTF8.ps1` 模组，在入口脚本中 dot-source 调用，一次配置所有编码变量。
- 如果团队混用 Windows PowerShell 5.1 和 PowerShell 7，明确基础设施只支持 UTF-8，并在 5.1 的 profile 中设置上述变量，降低分散配置的维护成本。
- 对于通过配置文件运行的工具（如某个 MCP 服务器），确保其 JSON 配置文件和所有输入数据都是 UTF-8 no BOM 或 UTF-8 with BOM（PowerShell 5.1 更推荐 BOM）。可以在部署脚本中增加编码检测和转换步骤。
- 全部 HTTP 调用行为统一采用显式 Content-Type + charset，不依赖默认推断。

---

## 总结

PowerShell 在 Windows 上“打坏”中文的本质，是编码假设链出现了断裂。只要把控制台、文件、网络三处的编码强制对齐到 UTF-8，并在每次数据边界上显式声明编码，这些乱码问题就会彻底消失。上述方法在大量实际自动化环境中已被验证稳定，无论你用的是 Windows PowerShell 5.1 还是 PowerShell 7，都可以立即复现和修复。希望这篇踩坑实录能让你少花几分钟在字符集堆里翻白眼，多一些时间专注于真正有价值的自动化逻辑。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-25/06bc0238049c0bb0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-25/2469ee5479d9270a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-25/d870030ebe0c04ef.png)

