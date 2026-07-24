---
title: Windows 中文 JSON API 调用：为什么 PowerShell 会把中文打坏
feedId: 30331
source: 综合讨论
publishedAt: 2026-07-25
---

## 背景

在 Windows 上做 OpenClaw 插件、MCP 工具或本地 Agent 时，用 PowerShell 调用 JSON API 是极为常见的模式。不管是 `Invoke-RestMethod` 拉配置、`Invoke-WebRequest` 抓数据，还是把结果交给下游程序，很多人都会遇到同一类问题：返回的 JSON 里明明有中文，一落地就变成乱码，或者直接被替换成问号。

这类问题往往不是 API 服务器的问题，而是 PowerShell 自身的编码行为在“帮忙”，尤其是在把输出写入文件、通过管道传给外部进程时。本文复现典型的编码损坏场景，给出可直接落地的排障思路与工程化建议，适用于所有依赖 PowerShell 做自动化的 Windows 环境。

## 问题复现

假设本地有一个返回中文 JSON 的测试 API（Flask/Express 均可），内容如下：

```json
{"status":"成功","message":"你好，世界"}
```

不少人会写出这样的 PowerShell 脚本：

```powershell
$resp = Invoke-RestMethod -Uri "http://127.0.0.1:5000/api/test"
$resp | ConvertTo-Json -Depth 5 > result.json
```

打开 `result.json`，看到的是：

```json
{
    "status":  "\u6210\u529f",
    "message": "\u4f60\u597d\uff0c\u4e16\u754c"
}
```

或者直接用 `Invoke-WebRequest` 保存内容：

```powershell
$resp = Invoke-WebRequest -Uri "http://127.0.0.1:5000/api/test"
$resp.Content > response.txt
```

打开 `response.txt` 发现中文变成 `????`。

更折磨人的是，在控制台直接打印 `$resp.status` 可能显示正常，但只要通过管道交出去，编码就崩了。此外，不同 PowerShell 版本（5.1 与 7+）、不同终端（Windows Terminal / conhost）表现还不一致。

## 原因分析

核心在于三个编码设置互相打架：

1. **重定向操作符 `>` 的默认编码**  
   在 Windows PowerShell 5.1 中，`>` 相当于 `Out-File -Encoding Unicode`，也就是 **UTF-16 LE**。如果下游程序只认 UTF-8，自然乱码。  
   PowerShell 7+ 的 `>` 默认改为 UTF-8（无 BOM），但很多旧脚本仍沿用 5.1 习惯。

2. **`ConvertTo-Json` 的转义行为**  
   Windows PowerShell 5.1 的 `ConvertTo-Json` 默认对非 ASCII 字符做 Unicode 转义（`\uXXXX`），而 PowerShell 7 引入了 `-EscapeHandling` 参数。如果你的 Agent 依赖可读的 JSON，转义后的字符串对日志和人工检查极不友好。

3. **控制台与管道编码分离**  
   `[Console]::OutputEncoding` 影响控制台显示，`$OutputEncoding` 影响管道传输到外部程序时的字节编码。默认值通常是 ASCII（代码页 1252），导致中文字节被截断或映射为 `?`。当 `Invoke-WebRequest` 返回的 `Content` 是基于服务器响应的 charset 解码过的字符串，再交给外部程序时，又会因为 `$OutputEncoding` 不匹配再次损坏。

## 工程化做法

针对上述三个根因，一条一条解决。

### 1. 使用 `Invoke-RestMethod` 时直接保存原始 JSON

如果需要把 API 返回的 JSON 原样落盘，不要走 `ConvertTo-Json` 再写文件的路径，直接用 `-OutFile` 参数：

```powershell
Invoke-RestMethod -Uri "http://127.0.0.1:5000/api/test" -OutFile result_raw.json
```

这样得到的是服务端响应的原始字节流，不经过 PowerShell 对象转换，字符编码由服务器宣告的 `Content-Type charset` 决定，通常是 UTF-8。

如果必须使用 `Invoke-WebRequest`，则从 `RawContentStream` 直接读取字节写入，避免字符串编码转换。

### 2. 强制统一编码

在脚本开头设置输出编码，并固定使用 `Out-File` 或 `Set-Content` 指定 UTF-8：

```powershell
# 控制台编码
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
# 管道编码
$OutputEncoding = [System.Text.Encoding]::UTF8

$response = Invoke-RestMethod -Uri "http://127.0.0.1:5000/api/test"
# 将对象序列化为 UTF-8 保存
$json = $response | ConvertTo-Json -Depth 10
$json | Out-File -Encoding UTF8 "result.json"
```

PowerShell 7 下可进一步控制 JSON 转义：

```powershell
$response | ConvertTo-Json -Depth 10 -EscapeHandling Default | Out-File -Encoding UTF8 "result.json"
```

`-EscapeHandling Default` 会保持非 ASCII 字符的字面形式，避免变成 `\uXXXX`。

### 3. 将结果传给外部程序时的管道处理

如果需要把 API 结果通过管道传给 Python、Node 等外部进程，一定要确保 `$OutputEncoding` 与目标进程的输入编码一致。例如：

```powershell
$OutputEncoding = [System.Text.Encoding]::UTF8
$json | python read_stdin.py
```

如果外部程序期望的是当前系统 OEM 编码（GBK），那 `$OutputEncoding` 也要相应调整。一般建议统一到 UTF-8，减少心智负担。

### 4. 处理 `Invoke-WebRequest` 的编码推断

某些 API 虽然返回 UTF-8 内容，但响应头未声明 `charset=utf-8`。此时 `Invoke-WebRequest` 可能按 ISO-8859-1 解码，导致 `$resp.Content` 已成烂字。可通过手动重读 RawContent 解决：

```powershell
$resp = Invoke-WebRequest -Uri "http://127.0.0.1:5000/api/test"
$raw = [System.Text.Encoding]::UTF8.GetString($resp.RawContentStream.ToArray())
```

## 踩坑点

- **PowerShell 版本差异**：`Out-File` 默认编码、`ConvertTo-Json` 的行为在 5.1 和 7+ 之间完全不同。生产环境建议锁定 PowerShell 7 并统一脚本风格。
- **BOM 问题**：`Out-File -Encoding UTF8` 默认带 BOM，某些 JSON 解析器（如 jq）对 BOM 敏感，可改用 `[System.IO.File]::WriteAllText("path", $json, [System.Text.UTF8Encoding]::new($false))` 输出无 BOM 的 UTF-8。
- **控制台字体**：即使用对编码，如果终端字体不支持中文，显示时仍会呈方块。这是显示问题，不是编码损坏，无需过度调试。
- **`curl.exe` 别名**：在 PowerShell 中 `curl` 默认指向 `Invoke-WebRequest`，而如果显式调用 `curl.exe`，则需要通过 `$OutputEncoding` 处理其 stdout 输出。

## 可复用建议

1. **模板化脚本头部**：在所有 PowerShell 自动化脚本开头注入：
   ```powershell
   $OutputEncoding = [Console]::OutputEncoding = [System.Text.Encoding]::UTF8
   ```
2. **持久化 API 结果时远离 `>`**：统一使用 `Out-File -Encoding UTF8` 或 `Set-Content -Encoding UTF8`。
3. **用对象，别用字符串**：尽量以对象形式在 PowerShell 内部传递 API 结果，仅在最终落盘或跨进程时序列化，这样能避免大部分中间编码转换。
4. **测试编码用 “中英文+emoji”**：Emoji 是高质量编码测试剂，如果 emoji 正常，中文通常也没问题（但需注意 UCS-2 与 UTF-16 的差异）。

## 总结

PowerShell 把中文 JSON “打坏”的根源，几乎总是默认编码不透明和版本间行为变化。对于构建在 Windows 上的 OpenClaw 工具链，脚本里多写一行编码声明，远比事后排查乱码节省时间。记住：**任何进入文件或管道的内容，都必须明确编码**——这条规则在自动化环境里再强调也不为过。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-25/93039fe94b2bc3b9.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-25/d739a8fc930eca10.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-25/aa96b68ecf9badc8.png)

