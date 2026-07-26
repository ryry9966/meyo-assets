---
title: PowerShell 调用中文 JSON API 乱码：一次排障与根治指南
feedId: 30543
source: 综合讨论
publishedAt: 2026-07-26
---

## 背景：在 Windows 自动化里踩中文字符的坑

最近在给 OpenClaw 的 Agent 配置几个本地 HTTP API，其中大部分返回中文内容（标题、用户名、地址等）。语言选型上，PowerShell 是 Windows 下最直接的「胶水」——Invoke‑RestMethod 配合 ConvertFrom‑Json 几行就能把结果拿回来。但调用完成后，控制台打印出的中文全是“????”，重定向到文件再用记事本打开，变成“ä½ å¥½……”，Agent 拿到的上下文自然跟着出问题。

这是 Windows 上 PowerShell 处理中文字符特有的编码连锁反应，并非 JSON 标准本身的问题，也不是某个库的 bug。只要搞清其中三层编码陷阱，根治并不复杂。下面是定位过程与一套可复用的工程化方案。

## 问题现象

场景：通过 `Invoke‑RestMethod` 调用一个返回 JSON 的接口，响应体示例为：

```json
{"city":"北京","temperature":"26°C"}
```

在 PowerShell 5.1 终端直接运行：

```powershell
$resp = Invoke-RestMethod -Uri "http://localhost:9000/weather" 
Write-Host $resp.city
```

控制台输出：`????` 或空白。若用 `> output.txt` 重定向，记事本打开显示类似 `äº¬` 的乱码，但用 VS Code 切换为 UTF‑8 编码后恢复正常。说明数据本身未损坏，只是显示与持久化环节编码错配。

## 原因剖析：三层编码陷阱

### 第一层：控制台宿主编码（OEM Code Page）

Windows 控制台默认代码页是 936（GBK）或 437 等 OEM 编码，而 PowerShell 内部字符串是 UTF‑16。当控制台宿主不支持宽字符转换时，输出中文就会降级为问号。即使你显式调用了 `chcp 65001`，当前脚本或模块可能没有持续生效，导致只在某个会话窗口内有效。

### 第二层：PowerShell 输出编码设置

PowerShell 有两个关键编码变量：
- `[Console]::OutputEncoding`：控制写入控制台输出流的编码，默认跟随系统区域设置（通常是 GBK 或 OEM）。
- `$OutputEncoding`：控制管道传递给外部程序时的编码，默认是 ASCII（US‑ASCII）。

对于 `Write‑Host` 和直接输出对象，`[Console]::OutputEncoding` 起作用。如果你把字符串用管道传给 `Out‑File` 或 `>` 重定向，又涉及 `$OutputEncoding` 和文件写入 cmdlet 的 `-Encoding` 参数。`Invoke‑RestMethod` 正确解析了 UTF‑8 的 JSON，将 `city` 属性存储为正常的 .NET 字符串，但当这个字符串流向显示或文件时，各环节默认编码串起来就把中文打坏了。

### 第三层：脚本文件本身的编码

PowerShell 5.1 要求脚本文件是 UTF‑16 LE 或 UTF‑8 with BOM 才能正确解析其中的中文字符串字面量。如果脚本保存为 UTF‑8 without BOM，中文注释或直接写在脚本内的中文可能就变成乱码，进而影响硬编码的 JSON 字符串测试。PowerShell 7+ 已默认支持无 BOM 的 UTF‑8，但在混合环境（5.1 和 7 并存）中仍易踩坑。

## 解决方案步骤（适用于 PowerShell 5.1 & 7）

以下步骤覆盖 99% 的控制台、重定向与文件写入场景，推荐在脚本开头或 `$PROFILE` 中统一设定。

**1. 强制控制台代码页为 UTF‑8**
```powershell
chcp 65001 > $null
```
在脚本内执行可确保当前控制台使用 UTF‑8。建议放入 `profile.ps1`，每次启动生效。

**2. 设置 PowerShell 输出编码**
```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$OutputEncoding = [System.Text.Encoding]::UTF8
```
第一条解决 `Write‑Host` 和对象直接输出的显示；第二条解决管道传递给外部程序（如 `python` 脚本）的中文传递。

**3. 使用 `Invoke‑RestMethod` 时明确字符集**
```powershell
$resp = Invoke-RestMethod -Uri $uri -ContentType "application/json; charset=utf-8"
```
虽然此 cmdlet 会自动检测 UTF‑8，但加上可读性更强，也能防止某些非标准响应头带来的误判。

**4. 写入文件显式指定编码**
```powershell
$resp.city | Out-File -FilePath "city.txt" -Encoding utf8NoBOM
# 或
$resp.city | Set-Content -Path "city.txt" -Encoding UTF8
```
`Out‑File` 默认是 Unicode（UTF‑16 LE），`Set‑Content` 默认是 ASCII？在 PowerShell 5.1 中默认可能是 ANSI，务必手动指定。推荐 `utf8NoBOM`，避免某些下游工具看到 BOM 误解析。

**5. 脚本文件保存为 UTF‑8 with BOM（仅限 PS5.1 场景）**
如果你还在维护 Windows PowerShell 5.1 项目，请用 VSCode 或记事本另存为“UTF‑8 with BOM”。PowerShell 7 用户无需 BOM。

## 踩坑实录

- **坑 1：`chcp 65001` 后仍然乱码**  
  检查是否在 ISE 或 VSCode 集成终端中运行。这些宿主可能绕过 `[Console]::OutputEncoding`。在 VSCode 中优先使用 `$psEditor` 或直接依赖 `Write‑Output` → 终端宿主自行处理。稳妥办法是避免依赖控制台输出可读性，Agent 取数据时直接操作对象属性。

- **坑 2：`ConvertTo‑Json` 再输出中文**  
  `ConvertTo‑Json` 默认会将中文字符转义为 `\uXXXX`，这不会乱码，但人类不易读。若想让 JSON 保持原始中文，可用 `ConvertTo‑Json -EscapeHandling EscapeNonAscii` 的相反参数？其实可用 `ConvertTo‑Json` 后直接输出到文件，不需要恢复为可读中文，Agent 处理转义序列也没有问题。但如果你确实要生成可读 JSON 文件，可结合 `[System.Web.HttpUtility]::JavaScriptStringEncode` 或直接拼接字符串，但工程上不建议牺牲正确性。

- **坑 3：`Invoke‑WebRequest` 的 `.Content`**  
  `Invoke‑WebRequest` 返回的 `.Content` 是字符串，已按响应头指定的编码解码。若服务端缺少 `charset`，可能错误地用 ISO‑8859‑1 解码，中文损坏。遇到这种情况，改用 `.RawContentStream` 并手动创建 `StreamReader` 指定 UTF‑8 读取。

- **坑 4：PowerShell 转义和 JSON 内部的双引号**  
  使用 `Write‑Host` 打印 JSON 字符串时，内部双引号若不正确转义，可能导致输出被截断，看起来像乱码，实际上是语法错误。避免 `Write‑Host` 直接打原始 JSON，改用 `$jsonString | Write‑Host` 或直接操作对象。

## 可复用工程建议

在为 OpenClaw Agent 编写 PowerShell 胶水脚本时，建议遵循以下模式：

- **环境统一入口**：在脚本公共模块或 `$PROFILE` 开头执行上述编码设置，并加上 `#requires -Version 7.0` 或注明最低版本，推荐逐步迁移到 PowerShell 7，避免 Windows 5.1 的遗留编码行为。
- **数据流不依赖控制台**：Agent 需要数据时，PowerShell 脚本应写入临时文件（JSON），并用 `Set‑Content -Encoding UTF8`，然后让 Agent 直接读取文件。避免 `StdOut` 解析，消除控制台编码依赖。
- **健壮性包装函数**：封装一个 `Invoke‑MyJsonApi` 函数，内部统一处理编码、异常和转换，输出对象而非字符串。后续直接访问对象属性，避免任何隐式编码转换。
- **调试快速检查**：遇到疑似乱码，立刻用 `[System.Text.Encoding]::UTF8.GetString([System.Text.Encoding]::Default.GetBytes($str))` 反向验证原始字节序列是否仍是正常的 UTF‑8，快速排除数据本身损坏的情况。

## 总结

Windows 上 PowerShell 把中文 JSON “打坏”，本质是控制台、输出编码与脚本文件 BOM 三层未对齐。这些不是 PowerShell 缺陷，而是 Windows 长久以来为了兼容性保留的 OEM/ANSI 代码页设计。一旦主动将这三层全部设置为 UTF‑8，中文 JSON 的调用、显示和落盘就完全正常，Agent 也就能拿到完整的语义内容。工程实践中，把编码设置沉淀为团队复用的脚本模板，能避免每次新开窗口都要重新排障。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-26/cdad02b87f46d16d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-26/aec7b6fa9310b52d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-26/1d52d7614c204e8f.png)

