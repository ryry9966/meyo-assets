---
title: PowerShell 调用 JSON API 时中文变“锟斤拷”的根因与工程化修复
feedId: 29347
source: 综合讨论
publishedAt: 2026-07-17
---

# 背景：当 Agent 的插件在 Windows 上吐出乱码

在 OpenClaw 社区里，越来越多的开发者用 PowerShell 写自动化脚本、MCP 工具或本地 Agent 插件。这类场景下，脚本需要通过 REST API 与外部系统交换数据——创建一个包含中文摘要的 Jira 工单、向企业微信发送通知、或者读取一份中文配置后 POST 给后端。然而很多人在 Windows 上第一次把中文塞进 `Invoke-RestMethod` 的 JSON body 时，会发现接收方看到的是 `锟斤拷`、`???` 或者整串 Unicode 转义字符（`\uXXXX`）。即便是 `-ContentType "application/json; charset=utf-8"` 写得明明白白，中文照样打坏。这个问题浪费过我不止一个下午，本文把根因和工程化做法一次性说清。

# 问题现象：同样的脚本，Linux 没问题，Windows 全乱套

最简单的复现代码：

```powershell
$body = @{ summary = "内存泄漏排查" } | ConvertTo-Json
Invoke-RestMethod -Uri http://localhost:8080/api/tasks -Method Post -Body $body -ContentType "application/json; charset=utf-8"
```

在 Windows PowerShell 5.1 下，后端收到的 body 可能是：

- `{"summary": "????"}`  
- `{"summary": "å
å­æ³„æ¼æŸ¥"}`  
- 或者直接抛出“Invalid JSON”错误，因为 PowerShell 把 `$body` 当作 UTF-16 LE 输出，而 `Invoke-RestMethod` 的默认编码行为不是预期的 UTF-8。

换成 PowerShell 7+，情况会好转，但重定向到文件、管道传递、或者在某些 codepage 非 65001 的控制台里，问题依旧阴魂不散。

# 根因：三层编码陷阱

### 1. `ConvertTo-Json` 的内存编码 vs 传递编码
`ConvertTo-Json` 生成的是 .NET 字符串（UTF-16 内存表示）。当你把它作为 `-Body` 参数直接传入时，PowerShell 需要把这个字符串编码成字节流发送。Windows PowerShell 5.1 下，`Invoke-RestMethod` 对 `-Body` 字符串的默认编码依赖 `[System.Text.Encoding]::Default`，也就是系统当前 ANSI 代码页（如 GBK/936）。如果本地 Windows 是中文系统，ANSI 是 GBK，而你的 JSON body 包含中文字符，它们会被按 GBK 编码发送，但 HTTP 头却写了 `charset=utf-8`。接收方按 UTF-8 解码，自然乱码。

PowerShell 7 改成了默认使用 UTF-8 without BOM，但如果你的脚本还需要兼容 5.1（很多企业环境依旧如此），就必须显式控制。

### 2. `$OutputEncoding` 和管道重定向
很多自动化脚本会把 API 返回的中文 JSON 传给下一个命令，例如：

```powershell
Invoke-RestMethod -Uri ... | Out-File result.json
```

`Out-File` 在 Windows PowerShell 5.1 默认使用 UTF-16 LE 编码，且 `$OutputEncoding` 默认是 ASCII。因此从 API 收到的 UTF-8 中文会被当成 ASCII 处理，导致写入文件时二次损坏。即便你显式指定 `-Encoding UTF8`，管道中的字符串已经被错误解码。

### 3. 控制台代码页与开发者体验
在 `cmd.exe` 或 VS Code 终端里，若代码页不是 65001（UTF-8），即使 PowerShell 内部处理正确，打印到屏幕也是乱码。这本身不直接影响 API 调用，但会严重干扰调试：你会误以为数据在发送前就坏了，从而做出错误修复。

# 工程化做法：从源头到终点全部锁定 UTF-8

### 步骤 1：强制发送 UTF-8 JSON body
不要依赖隐式转换，直接构造字节数组：

```powershell
$payload = @{ summary = "内存泄漏排查" } | ConvertTo-Json
$utf8Bytes = [System.Text.Encoding]::UTF8.GetBytes($payload)
Invoke-RestMethod -Uri http://localhost:8080/api/tasks -Method Post -Body $utf8Bytes -ContentType "application/json; charset=utf-8"
```

传给 `-Body` 一个 `[byte[]]`，PowerShell 会原样发送，完全绕开字符串编码猜测。

### 步骤 2：读取响应内容时正确解码
如果 API 返回 `byte[]`，用 `[System.Text.Encoding]::UTF8.GetString()` 解码。如果直接得到字符串，检查响应头是否指定了正确的 `charset`；同时确保 `$OutputEncoding` 已设好，以免后续管道操作出错：

```powershell
$OutputEncoding = [System.Text.Encoding]::UTF8
```

这条设置对 Windows PowerShell 5.1 尤其重要。

### 步骤 3：统一脚本文件的编码
将 `.ps1` 文件另存为 **UTF-8 with BOM**（Windows PowerShell 5.1 要求 BOM 以正确读取脚本中的中文），或者在文件头加上 `# -*- coding: utf-8 -*-`（PowerShell 6+ 支持）。同时推荐在脚本开头显式声明：

```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$OutputEncoding = [System.Text.Encoding]::UTF8
```

### 步骤 4：验证请求体
在调试阶段，可以用 `Invoke-WebRequest` 或抓包工具确认实际发送的字节：

```powershell
$utf8Bytes | ForEach-Object { '{0:X2}' -f $_ } -join ' '
```

对照 JSON 中的中文字节查看是否为正确的 UTF-8 序列（例如 `内` 的 UTF-8 为 `E5 86 85`）。

# 踩坑记录：几个常见但隐蔽的错误

- **在 PowerShell 5.1 中使用了 `ConvertTo-Json -Compress` 然后直接当字符串发**：问题依旧，压缩并不改变编码。
- **用 `curl.exe` 代替 `Invoke-RestMethod`**：如果命令行参数含中文，curl 的行为依赖于当前 codepage，反而更不稳定。除非你能保证 `chcp 65001` 且所有参数正确转义。
- **用 `@()` 或 `[ordered]@{}` 构造复杂 JSON 后直接发送**：一样的字符串传递问题，必须转成字节。
- **跨进程管道**：比如脚本结果通过 stdin 传给 Python/Node 进程，又写回文件。Windows 下管道编码默认 OEM，要特别小心。尽量通过 HTTP 或文件交互，且文件明确用 UTF-8 without BOM。

# 可复用建议

1. **封装安全模块**  
   在团队内部打造一个 `Invoke-SafeRestMethod`，内部强制将 body 转为 `[byte[]]`，并预先设置编码。从此业务脚本不再关心编码细节。

2. **兼容性检查**  
   若脚本需同时跑在 Windows PS 5.1 和 PS 7 上，在开头加上：

   ```powershell
   if ($PSVersionTable.PSVersion.Major -lt 6) {
       $OutputEncoding = [System.Text.Encoding]::UTF8
       [Console]::OutputEncoding = [System.Text.Encoding]::UTF8
   }
   ```

3. **CI/CD 中验证**  
   在自动化测试里加入一条“发送含固定中文的 JSON 并校验返回内容是否一致”的用例，把它作为部署前检查项，可提前拦住编码回归。

# 总结

PowerShell 在 Windows 上把中文 JSON 打坏，不是语言本身的问题，而是历史遗留的编码默认值与现代 API 实践之间的冲突。只要记住：**从内存到网络、从网络到文件，所有环节显式指定 UTF-8，且传递时使用字节数组避免隐式转换**，这个问题就能被彻底根治。对 OpenClaw 生态内编写插件或 MCP 工具的开发者而言，这层编码归一化能省去大量和“锟斤拷”搏斗的无效时间，让你专注在逻辑本身。

---

