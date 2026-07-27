---
title: Windows 中文 JSON API 调用：为什么 PowerShell 会把中文打坏？
feedId: 30711
source: 综合讨论
publishedAt: 2026-07-28
---

# Windows 中文 JSON API 调用：为什么 PowerShell 会把中文打坏？

## 背景

在 OpenClaw、MCP 服务或自动化管线的实践中，我们经常用 PowerShell 脚本充当“胶水层”，快速调用 RESTful API。例如：Agent 需要把一段中文摘要推送到内部平台，或者插件通过 Invoke-RestMethod 提交带中文关键字的查询。这类脚本在开发机（Windows）上经常遇上一个经典故障：**中文 JSON 到了服务端就变成一堆乱码，或者返回的中文在控制台里显示为“????”。** 这不是偶然的 bug，而是 Windows 下 PowerShell 的编码历史债务与现代 API 的 UTF-8 约定相撞的结果。

本文通过一个最小化复现场景，拆解编码断裂点，给出可在工程中直接复用的修复方案与踩坑记录。

## 问题复现：发送一个中文 JSON 对象

假设我们有一个本地测试 API（例如 FastAPI 的 POST /echo），它原样返回收到的 JSON。我们用 PowerShell 7（或 5.1）写下最简单的调用：

```powershell
$body = @{ content = "你好，自动化管线" } | ConvertTo-Json
Invoke-RestMethod -Uri "http://127.0.0.1:8000/echo" -Method Post -Body $body -ContentType "application/json"
```

看到的返回结果中，字段内容变成了 `ä½ å¥½ï¼Œè‡ªå¨åŒ–ç®¡çº¿` 或一堆问号。如果查看服务端日志，接收到的原始字节其实是 Windows-1252 或 Latin-1 编码的产物，而不是期望的 UTF-8。

## 为什么会被“打坏”？

根本原因在于 **PowerShell 在将字符串转换为网络传输字节流时，所使用的编码与 API 端期望的 UTF-8 不一致**。这个不一致来自多个层次的叠加：

1. **`-Body` 字符串的序列化默认编码**  
   在 Windows PowerShell 5.1 中，当 `-Body` 参数是字符串且 `-ContentType` 未显式指定 `charset` 时，`Invoke-RestMethod`/`Invoke-WebRequest` 会将字符串按 **ISO-8859-1**（Latin-1）编码发送。这意味着所有非 Latin-1 字符（包括中文）会被破坏。  
   PowerShell 7 改善了这一点：字符串 body 默认使用 **UTF-8 无 BOM** 发送。但很多生产环境仍在使用 5.1，或者遇到接收端要求明确 `charset=utf-8` 的情况。

2. **控制台输出编码**  
   即使发送方正确，API 返回的中文 JSON 在控制台显示时还可能被再次破坏。因为 `[Console]::OutputEncoding` 在中文 Windows 上常常是 OEM 代码页（如 936），无法容纳返回的 UTF-8 字节序列，导致字符被替换为 “?”。

3. **文件/脚本本身编码**  
   如果在脚本里直接写死中文字符串（如 `$body = '{"msg":"你好"}'`），而脚本文件不是 UTF-8 with BOM 保存（PS 5.1 的要求），那么脚本在被解析时就已经破坏了中文。这个隐性因素会让一切后续修复手段都失效。

## 做法/步骤：从断裂到稳定

下面给出一个稳定的调用模式，兼容 Windows PS 5.1 与 PS 7，无需迷信第三方便宜解决方案。

**1. 保证脚本文件编码**  
用 VS Code 等编辑器将 `.ps1` 文件保存为 **UTF-8 with BOM** 编码。如果必须使用无 BOM（常用于 Git 管理），可在脚本开头强制声明编码：

```powershell
$PSDefaultParameterValues['*:Encoding'] = 'utf8'
```

但更彻底的还是在 PS 5.1 中使用 BOM。

**2. 显式指定字符集**  
调用时将 `Content-Type` 写全：

```powershell
$contentType = "application/json; charset=utf-8"
```

这能促使某些底层处理自动选择正确编码。

**3. 以字节方式发送 body**  
最可靠的方式是让 PowerShell 不替我们解释字符串，直接发送 UTF-8 字节。可以这样：

```powershell
$payload = @{ content = "你好，自动化管线" } | ConvertTo-Json -Compress
$bodyBytes = [System.Text.Encoding]::UTF8.GetBytes($payload)
Invoke-RestMethod -Uri $url -Method Post -Body $bodyBytes -ContentType "application/json; charset=utf-8"
```

注意这里 `-Body` 收到一个字节数组时，`Invoke-RestMethod` 会直接原样发送，不再额外编码。

**4. 修复控制台显示**  
为避免返回的中文在终端乱码，设置输出编码为 UTF-8：

```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$OutputEncoding = [System.Text.Encoding]::UTF8
```

`$OutputEncoding` 影响管道和重定向，`[Console]::OutputEncoding` 影响控制台直接输出。两个都设上，避免疏漏。

**5. 验证整个通道**  
在脚本中，可以在发送前用 `[System.Text.Encoding]::UTF8.GetString($bodyBytes)` 回显检查字节是否完好；在接收后用 `$response.content` 直接查看，确认显示无误。

## 踩坑点

- **curl 别名陷阱**：Windows 自带 `curl.exe` 并不同于 Unix curl 的编码行为，且 PowerShell 的 `curl` 别名指向 `Invoke-WebRequest`。在脚本中使用 `curl` 容易混淆期待。统一用 `Invoke-RestMethod` 或 `Invoke-WebRequest`，并且不要假设别名行为。
- **PS 5.1 下 `ConvertTo-Json` 的深度与编码**：`ConvertTo-Json` 默认深度只有 2，复杂嵌套对象会丢失。且序列化结果在 5.1 中可能带有不必要的空格和换行；建议使用 `-Compress` 并手动设置深度 `-Depth 4` 以上。更重要的是，序列化输出的字符串在内存中正确，但前面提到的 body 编码问题仍是单独的问题。
- **BOM 问题不只是脚本**：API 返回的 JSON 如果带有 BOM，虽然 Unicode 标准允许但很多解析器会报错。如果遇到 `Invoke-RestMethod` 返回“unexpected token”，可用 `-ResponseEncoding utf8` 或自己处理原始字节去除 BOM。
- **环境变量与代码页**：`chcp 65001` 仅在当前 cmd 会话生效，对 PowerShell 子进程不稳定。用 .NET 的 `[Console]::OutputEncoding` 更可靠。

## 可复用建议

把这些设置整合成一个所有外部调用脚本共用的 **“安全外壳”**：

```powershell
# 安全编码预设
$OutputEncoding = [Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$PSDefaultParameterValues['*:Encoding'] = 'utf8'

function Invoke-SafeRestMethod {
    param($Uri, $Method = 'Post', $BodyObject)
    $json = $BodyObject | ConvertTo-Json -Compress -Depth 10
    $bytes = [Text.Encoding]::UTF8.GetBytes($json)
    Invoke-RestMethod -Uri $Uri -Method $Method -Body $bytes -ContentType "application/json; charset=utf-8"
}
```

这样在 Agent 插件中调用任何内部 API，只需传入哈希表，编码问题就被控制在一个函数内部。OpenClaw 用户可以将该函数放在 `profile.ps1` 或共用模块中，确保整条自动化管线不再被“中文打坏”。

## 总结

PowerShell 在 Windows 上调用中文 JSON API 乱码，不是某个命令的 bug，而是**历史遗留的编码假设与现代 Web 标准不一致**的系统性问题。关键修复点在于：

- 用字节流发送 body 而不是依赖隐式编码；
- 在 Content-Type 中标明 `charset=utf-8`；
- 将脚本文件保存为 UTF-8 with BOM（PS 5.1）；
- 同时设置 `[Console]::OutputEncoding` 和 `$OutputEncoding` 为 UTF-8。

一旦将这些措施固化为团队约定，就能在 Windows 上构建健壮的中文自动化管道，不再为乱码消耗排查时间。下一次你的 Agent 发出“你好”时，API 一定收到准确的问候。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-28/e535b5649aaa5ed3.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-28/3afe7333700dea0f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-28/3a24a8808fbf01c8.png)

