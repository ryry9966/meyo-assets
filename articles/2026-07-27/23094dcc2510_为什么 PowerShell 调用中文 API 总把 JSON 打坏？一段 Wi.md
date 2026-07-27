---
title: 为什么 PowerShell 调用中文 API 总把 JSON 打坏？一段 Windows 编码自救指南
feedId: 30667
source: 综合讨论
publishedAt: 2026-07-27
---

## 背景：一份中文 JSON 从正确到乱码，只需要一次 PowerShell 管道

在 Windows 上做自动化、接 MCP 插件或喂数据给 Agent 时，很多同学习惯直接在 PowerShell 里用 `curl` 或 `Invoke-WebRequest` 调一个返回中文的 REST API，然后把响应丢给下游解析。例如：

```powershell
$resp = curl http://127.0.0.1:8080/api/items
$resp.Content | Out-File result.json
```

结果下游的 JSON 解析器（无论是 Python 脚本、MCP 工具还是 OpenClaw 的 skill）直接报错，或者解析出“锟斤拷”“烫烫烫”。看上去像是 API 返回了乱码，实际上 API 服务本身正常，是 **PowerShell 在管道传输和文件写入过程中悄悄改了编码**。

在 Agent 链路里，这种问题隐蔽性极强——中间某个环节的返回值一旦乱码，后续结构化提取、自然语言推理全部失效，排查时又容易归咎于网络或服务端。其实大部分情况只和 Windows 下 PowerShell 的默认编码行为有关。

---

## 问题：为什么 PowerShell 会“打坏”中文 JSON？

核心原因是 **Windows PowerShell 5.1 的默认编码不是 UTF-8**。

1. **管道输出的重编码**
   当你在 PowerShell 5.1 中运行 `curl`（实际上是 `Invoke-WebRequest` 的别名）获取响应体时，返回的 `Content` 属性已经是字符串。这个字符串在转换时，PowerShell 会使用当前控制台的输出编码（通常是系统 OEM 代码页，如 GBK/936），而不是响应头里声明的 UTF-8。于是中文字节被按单字节扩展解释，产生不可逆的损坏。

2. **文件写入的默认编码**
   `Out-File` 和重定向运算符 `>` 的默认编码是 **UTF-16 LE**（Unicode），不是 UTF-8。即便你成功拿到正确的 UTF-8 字符串，直接 `Out-File` 也可能生成带有 BOM 或两字节编码的文件，导致其他 UTF-8 工具读取失败或看到多余字符。

3. **`curl.exe` 与别名的坑**
   不少人会特意删除 `curl` 别名去调用真正的 `curl.exe`，但这依然可能掉进控制台编码陷阱：`curl.exe` 输出 UTF-8 字节流，PowerShell 捕获后仍会用 `[Console]::OutputEncoding` 解码为字符串，乱码照旧产生。

4. **PowerShell 7 的改进**
   PowerShell Core (7+) 则将默认编码全面切为 UTF-8，且 `Out-File` 和 `Set-Content` 在不指定 `-Encoding` 时也默认使用 UTF-8 无 BOM，问题大幅减少。但很多 Windows 生产环境仍内置 5.1 版本。

---

## 做法 / 步骤：从源头到文件的全链路 UTF-8 化

这里给出的方案同时适用于 Windows PowerShell 5.1 和 7+，确保 Agent 链路上的中文 JSON 再不乱码。

### 1. 强制 $OutputEncoding 与控制台编码

在脚本最前面加入：

```powershell
$OutputEncoding = [System.Text.UTF8Encoding]::new($false)
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
```

`$OutputEncoding` 影响命令输出到管道时的编码，`[Console]::OutputEncoding` 影响控制台程序（如 `curl.exe`）输出的解码方式。这两行是解决乱码的第一步。

### 2. 用 `Invoke-RestMethod` 直接拿到对象，而不是字符串

如果 API 返回的是 `application/json`，最优解是不接触原始字节，直接用：

```powershell
$data = Invoke-RestMethod -Uri "http://127.0.0.1:8080/api/items" -ContentType "application/json; charset=utf-8"
```

`Invoke-RestMethod` 会自动解析 JSON 为 PowerShell 对象，不会发生字符串级编码错误。后续直接使用对象属性，或者用 `ConvertTo-Json -Depth 10` 转回 JSON 字符串，也能保证编码正确。

> 但注意：如果你的下游需要一个**原始 JSON 文件**，那 `ConvertTo-Json` 再写文件时仍要指定编码（见第4步）。

### 3. 需要原始响应流时，手动解码

如果必须拿到原始字节流再自己处理（例如响应 Content-Type 不标准），不要用 `.Content`，用 `.RawContentStream`：

```powershell
$resp = Invoke-WebRequest -Uri "http://127.0.0.1:8080/api/items"
$reader = [System.IO.StreamReader]::new($resp.RawContentStream, [System.Text.Encoding]::UTF8)
$jsonString = $reader.ReadToEnd()
$reader.Close()
```

这样可以完全旁路 PowerShell 的自动编码猜测。

### 4. 文件写入一律显式指定 UTF-8

任何写文件操作都加上 `-Encoding utf8`（注意是小写的 `utf8`）：

```powershell
$jsonString | Out-File -FilePath result.json -Encoding utf8
```

或使用 `Set-Content`：

```powershell
Set-Content -Path result.json -Value $jsonString -Encoding utf8
```

如果必须使用重定向，先放弃这个习惯，用 cmdlet 替代。

### 5. 脚本级「UTF-8 保险」

如果是 .ps1 脚本，推荐在文件开头加上 `#requires -Version 5.1` 确保版本被识别，并统一设置编码。如果允许依赖，直接使用 PowerShell 7 是终结方案。

---

## 踩坑点：那些你以为已经解决的情况

- **`Out-File -Encoding utf8` 仍然写 BOM**  
  PowerShell 5.1 的 `-Encoding utf8` 会输出 **带 BOM 的 UTF-8**。大多数现代工具不介意，但部分 JSON 解析器（如严格模式的 `jq`）会报错。如果真的需要无 BOM，可以使用 `[System.IO.File]::WriteAllText("result.json", $jsonString, [System.Text.UTF8Encoding]::new($false))`。

- **PowerShell 内嵌 `cmd /c` 或调用外部 CLI**  
  在 MCP server 实现里，如果通过 PowerShell 启动外部 Python 脚本，也要确保 Python 进程输出被正确解码。最佳实践是在 Python 里显式 `sys.stdout.reconfigure(encoding='utf-8')`，并在 PowerShell 侧同步设置 `$env:PYTHONIOENCODING='utf-8'`。

- **控制台字体与显示乱码不要混淆**  
  有时只是控制台字体不支持显示中文，但字节本身正确。测试是否真的乱码可以用 `Get-Content result.json -Encoding utf8` 看输出，或者用 `certutil -encodehex` 检查十六进制：中文正确时应该看到连续的 `E4`、`B8`、`AD` 等三字节 UTF-8 序列。

- **MCP 工具链中的二次编码**  
  如果你把 JSON 通过标准输出返回给 MCP client（比如 OpenClaw 的本地执行器），务必确认整个通道的编码一致。在编写 MCP 插件的 `server.ps1` 时，同样需要使用上述配置，并显式用 `Write-Output` 输出 UTF-8 字符串，不要依赖于控制台重定向。

---

## 可复用建议：一个标准模板

无论是日常调试，还是给 Agent 做系统级配置，可以用下面这一小段作为所有 PowerShell HTTP 交互的头部模板：

```powershell
# Start-Utf8Environment
$OutputEncoding = [System.Text.UTF8Encoding]::new($false)
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
$ProgressPreference = 'SilentlyContinue'  # 加速下载

# 示例：安全获取中文 JSON
$json = Invoke-RestMethod -Uri $apiUrl -ContentType "application/json; charset=utf-8"
$json | ConvertTo-Json -Depth 10 | Set-Content -Path output.json -Encoding utf8
```

把它封装成一个 `Set-Utf8Environment` 函数，或者放到你的 `$PROFILE` 里，可以一劳永逸避免大部分乱码。

---

## 总结

Windows 下 PowerShell 处理中文 API 的乱码问题，**不是 API 的锅，不是网络的锅，是默认编码的锅**。其根因是系统沿用老旧的 OEM 代码页，与 UTF-8 生态产生冲突。在 Agent 和自动化链路中，任何一次非显式编码转换都可能让数据不可逆地损坏。

解决路径非常明确：  
1. 设置 `$OutputEncoding` 和 `[Console]::OutputEncoding` 为 UTF-8；  
2. 优先用 `Invoke-RestMethod` 拿对象而非字符串；  
3. 文件写入永远显式指定 `-Encoding utf8`；  
4. 如果环境允许，迁移到 PowerShell 7 是性价比最高的方案。

做到这几点，你的中文 JSON 就能在 Windows 自动化流水线里丝滑传递，不再让 Agent 卡在“烫烫烫”上。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-27/116e39bd2398be5b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-27/08c47e7c94fd5aca.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-27/a8921ed08999ce3b.png)

