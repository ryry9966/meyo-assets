---
title: Windows 中文 JSON API 调用：为什么 PowerShell 会把中文“打坏”
feedId: 30572
source: 综合讨论
publishedAt: 2026-07-26
---

## 1. 背景：PowerShell 在自动化链路中的位置

在 OpenClaw、MCP 插件、各类 Agent 工作流的日常实践中，Windows 环境下的自动化脚本经常通过 PowerShell 来调用 JSON API。无论是通过 `Invoke-RestMethod` 拉取配置、上报状态，还是用 `ConvertTo-Json` 准备数据包，PowerShell 都承担着「数据搬运工」的角色。

这类场景有一个共同特征：上下文里必然会带上中文，比如用户姓名、地区说明、系统日志输出。一旦中文在管道里“被打坏”，下游的 Agent 就会收到乱码，导致逻辑错乱、存储异常，甚至会让整个自动化链路静默失败。

本篇文章围绕一道高频问题展开：**为什么 PowerShell 在处理包含中文的 JSON 时，输出会出现乱码？以及如何在工程中稳定避开这个坑。**

## 2. 问题复现：看得见的异常

在 Windows 10/11 的 PowerShell 5.1 环境下，执行以下代码就能触发问题：

```powershell
$body = @{ message = "操作成功" } | ConvertTo-Json
Invoke-RestMethod -Uri http://localhost:8080/api/log -Method Post -Body $body
```

API 服务端收到的内容可能是 `"message":"操作成功"`，也可能是 `"message":"æ“ä½œæˆåŠŸ"`、`"message":"鎿嶄綔鎴愬姛"`，甚至出现 `���`。更隐蔽的情况是，直接在控制台输出 JSON 看起来正常，但一旦通过 `>` 重定向到文件，或通过 MCP server 的标准输出传递出去，文件或下游进程里就变成了 UTF-16 LE 编码的二进制 dump，中文完全不可读。

还有一种典型场景：使用 `Invoke-WebRequest` 获取远程 API 返回的 JSON，响应的 body 中中文正常，但当你用 `ConvertFrom-Json` 解析后重新序列化再发送，编码就坏了。

## 3. 根因分析：两套编码体系的冲突

问题核心不是 PowerShell“不支持”UTF-8，而是它在不同宿主、不同输出目标下，会静默切换编码策略。

### 3.1 `$OutputEncoding` 与 `[Console]::OutputEncoding`

- `[Console]::OutputEncoding` 决定控制台显示时的编码，默认是系统 OEM 代码页（如简体中文 Windows 上是 GBK/936）。
- `$OutputEncoding` 用于管道、重定向及向外部程序发送数据时的编码，**默认是 ASCII（US-ASCII）**。

当你用 `>` 把 JSON 流重定向到文件时，PowerShell 会用 `$OutputEncoding` 来编码文本，ASCII 无法表示中文字符，于是直接替换为 `?` 或乱码。更糟的是，很多第三方模块在内部启动子进程时，也用 `$OutputEncoding` 作为通信编码。

### 3.2 `ConvertTo-Json` 的默认行为

PowerShell 5.1 的 `ConvertTo-Json` 会把非 ASCII 字符转义为 `\uXXXX` 形式，看似避免了编码问题，实则引入另一个坑：在很多 API 或 MCP 插件里，后端期望的是原生的 UTF-8 中文，而不是转义序列。如果你为了保持可读性强行用 `-Compress` 或自行替换，又容易踩到 `System.Text.Encoding` 的设置。

### 3.3 外部程序与字节流的本质

`Invoke-RestMethod` 和 `Invoke-WebRequest` 发送 `-Body` 时，参数如果是字符串，PowerShell 会使用 `$OutputEncoding` 将其转为字节流。如果 `$OutputEncoding` 不是 UTF-8，中文二次编码就坏了。即使先转成字节数组 `[System.Text.Encoding]::UTF8.GetBytes($body)`，也要显式传入 `-Body`，而不是依赖默认行为。

## 4. 工程化解决方案（步骤）

### 4.1 强制统一 UTF-8，消灭隐式转换

在每个用到中文的脚本顶部添加：

```powershell
$OutputEncoding = [Console]::InputEncoding = [Console]::OutputEncoding = New-Object System.Text.UTF8Encoding $false
```

> 注意 `UTF8Encoding $false` 避免写文件时自动带上 BOM（Byte Order Mark），很多 API 网关、JSON 解析器会因 BOM 而报错。

### 4.2 发送 JSON body 时，显式指定字节流和 Content-Type

不要只传字符串，而是传一个字节流，并明确编码：

```powershell
$bodyJson = $body | ConvertTo-Json -Depth 5 -Compress
$utf8Bytes = [System.Text.Encoding]::UTF8.GetBytes($bodyJson)
$response = Invoke-RestMethod -Uri $uri -Method Post -Body $utf8Bytes -ContentType "application/json; charset=utf-8"
```

如果使用的是 `Invoke-WebRequest`，同样的原则适用。这样做彻底绕过 `$OutputEncoding` 的影响。

### 4.3 重定向到文件时，使用 `Out-File` 或 `Set-Content` 并指定编码

替换危险的 `>` 重定向：

```powershell
$json | Out-File -FilePath data.json -Encoding utf8NoBOM
# 或者
$json | Set-Content -Path data.json -Encoding UTF8
```

在 PowerShell 5.1 中，`UTF8` 编码会带 BOM。如果坚持不要 BOM，可以使用：

```powershell
$Utf8NoBomEncoding = New-Object System.Text.UTF8Encoding $false
[System.IO.File]::WriteAllLines("data.json", $json, $Utf8NoBomEncoding)
```

### 4.4 解析从外部收到的 JSON 时，提前转码

从 REST API 或子进程中拿到的乱码字符串，用以下方式抢救：

```powershell
$bytes = [System.Text.Encoding]::GetEncoding("iso-8859-1").GetBytes($malformedString)
$correctStr = [System.Text.Encoding]::UTF8.GetString($bytes)
```

这是经典的“UTF-8 字节被按 Latin-1 解释”的修复方式，适用于 Windows 环境下很多类似病灶。

## 5. 踩坑点与高阶排错

### 5.1 VS Code / Windows Terminal 与 PowerShell ISE 的行为差异

很多开发者在 VS Code 或 Windows Terminal 中测试正常，一旦放到任务计划或 CI agent 的 `powershell.exe` 下就炸了。原因在于现代终端默认会把 `[Console]::OutputEncoding` 设为 UTF-8，而裸 `powershell.exe` 仍然是 OEM 代码页。一定要在脚本内部显式设置，不要依赖宿主。

### 5.2 `ConvertTo-Json` 的深度与转义陷阱

默认 `-Depth` 为 2，容易被忽略，导致复杂对象序列化不全，产生被截断的 JSON。同时 PowerShell Core (7+) 中 `ConvertTo-Json` 默认不再转义非 ASCII，如果要兼容旧版，务必统一使用强制的 UTF-8 字节流方案，避免因版本差异导致线上故障。

### 5.3 日志输出中对编码的二次伤害

OpenClaw 这类框架中，模块通过 stdout 向 Agent 返回结果。如果模块内使用 `Write-Output` 打印包含中文的 JSON，而外部读取 stdout 时编码设置不对，就会乱码。建议模块内部统一用 `[Console]::OpenStandardOutput()` 直接写字节流，或至少在最外层用 `$OutputEncoding = [System.Text.Encoding]::UTF8` 锁死编码。

## 6. 可复用建议

- **封装一个安全 JSON 发送函数**，统一处理编码和 contentType，所有自动化脚本复用。
- **编写模块级初始化脚本**，在所有 `psm1` 入口设置 `$OutputEncoding`，避免调用者忘记。
- **MCP 插件作者注意**：接收 stdin 数据时，明确使用 UTF-8 读取，例如 `$inputJson = [System.IO.StreamReader]::new([Console]::OpenStandardInput(), [System.Text.Encoding]::UTF8).ReadToEnd()`。
- **添加 CI 检查**：在测试阶段用 ASCII 无效字符检测脚本，确保生成的 JSON 文件不含非预期编码痕迹。

## 7. 总结

PowerShell 把中文 JSON“打坏”的本质，是 PowerShell 在 Windows 旧版兼容性包袱下，对数据输出流和管道编码管理不够透明。一旦你理解了 `$OutputEncoding`、控制台编码与字节流之间的关系，问题便退化为一把可以拧紧的螺丝。

在大规模自动化与 MCP 生态中，稳定的数据处理比任何炫技都重要。不论你是给 OpenClaw 写插件，还是在 Agent 里串联多个 API，记得永远不要信任默认编码，永远把自己想象成管道中最“憨”但最可靠的网关——把中文安全地从 A 点运到 B 点，没有任何惊喜。

---

