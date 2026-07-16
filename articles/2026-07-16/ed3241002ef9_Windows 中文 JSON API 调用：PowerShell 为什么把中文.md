---
title: Windows 中文 JSON API 调用：PowerShell 为什么把中文打坏，以及工程化修复
feedId: 29305
source: 综合讨论
publishedAt: 2026-07-16
---

## 1. 背景：Agent、MCP 与自动化里的那个隐秘角落

当你在 Windows 上为 OpenClaw 写 Agent、构建 MCP 工具或自动化流水线时，大概率会用到 PowerShell 来调用 REST API。尤其是那些需要将结果喂给 LLM 的场景，JSON 里一旦出现中文，踩坑的概率就直线上升——你明明在终端里看到正确的中文，但落盘的文件或管道给下游的程序时，却变成了一堆 `????` 或 `ç¿»è¯‘` 这类“乱码纹”。

这个问题并不新鲜，但在 Agent 工程化实践中杀伤力极大：它让你怀疑自己的代码、怀疑 API 返回、怀疑网络，最后发现只是 PowerShell 的编码行为跟你开了个跨世纪的玩笑。下面我会用最简单的例子复现、拆解，并给出可复用的工程化方案。

## 2. 问题：能看不能用的中文 JSON

假设我们有一个返回中文的 JSON API，例如一个本地 mock：

```json
{ "message": "你好，OpenClaw 已经连接" }
```

用 PowerShell 5.1 调一下，并把结果保存到文件：

```powershell
$resp = Invoke-RestMethod -Uri "http://localhost:8080/api/hello"
$resp | ConvertTo-Json | Out-File -FilePath "result.json"
```

打开 `result.json`，你可能看到：

```json
{
    "message":  "??, OpenClaw ?????"
}
```

此时你再用 Python 或 OpenClaw 读取这个文件，得到的全是损坏的 Unicode。更诡异的是，如果直接在控制台执行 `Write-Host $resp.message`，中文显示正常。这种“看得到却存不住”的现象，根源全在编码。

## 3. 原因：三套编码体系的同时背叛

PowerShell 5.1（Windows 内置版本）在以下三个环节有各自的默认编码：

- **控制台输出**：`[Console]::OutputEncoding` 默认是系统的 OEM 代码页（如 936 简体中文），所以 `Write-Host` 中文能正常显示。
- **重定向运算符 `>`**：输出到文件时，使用 `$OutputEncoding` 变量，但它实际影响的是与外部程序交互的编码，内部重定向结果往往是 **UTF-16 LE**（Unicode），而不是你期望的 UTF-8。
- **`Out-File` / `Set-Content` 无编码参数时**：`Out-File` 默认使用 **Unicode (UTF-16 LE)**；`Set-Content` 默认使用 **ANSI 代码页**（如 Windows-1252），两者都不兼容 UTF-8 中文。

更隐秘的是，`Invoke-RestMethod` 返回的是解析好的 PSCustomObject，`ConvertTo-Json` 再序列化时，字符串已经是 .NET 内部 Unicode 表示，并不会自动帮你加上 UTF-8 的标记。当你用默认行为写入文件时，编码灾难就发生了。

## 4. 可靠做法（附可直接拷贝的代码）

### 4.1 显式指定 UTF-8 编码

最彻底的方案：在每一步输出都带上 `-Encoding utf8`。

```powershell
$resp = Invoke-RestMethod -Uri "http://localhost:8080/api/hello"
$json = $resp | ConvertTo-Json -Compress
$json | Out-File -FilePath "result.json" -Encoding utf8
```

或使用 `Set-Content`：

```powershell
$json | Set-Content -Path "result.json" -Encoding utf8
```

两种写法的文件最终都是合法的 UTF-8（不带 BOM），后续工具都能正确读取。

### 4.2 使用 Invoke-RestMethod 的 -OutFile 参数

如果你只是想把 API 原始响应存成文件，根本不需要对象解析和二次序列化：

```powershell
Invoke-RestMethod -Uri "http://localhost:8080/api/hello" -OutFile "result.json"
```

此时文件内容是 API 返回的原始字节流，编码由服务端决定。只要服务端返回的是 UTF-8 JSON，文件就是正确的。这是最安全、最少环节的方式。

### 4.3 修改脚本级默认编码（仅限 PowerShell 5.1 的妥协方案）

在脚本开头加上：

```powershell
$PSDefaultParameterValues['Out-File:Encoding'] = 'utf8'
$PSDefaultParameterValues['*:Encoding'] = 'utf8'  # 谨慎使用
```

这可以避免漏写 `-Encoding`，但需要团队成员都清楚这一全局修改，推荐只用在固定用途的脚本中。

### 4.4 终极方案：切换到 PowerShell 7+

在 PowerShell 7 (pwsh) 中，`Out-File` 和 `Set-Content` 的默认编码已改为 **UTF-8 无 BOM**，且重定向 `>` 也输出 UTF-8。如果你的 Agent 运行环境可以安装 pwsh，直接迁移能一劳永逸解决 90% 的编码问题。

## 5. 踩坑点实录

- **`>` 重定向的 UTF-16 LE 陷阱**  
  `Invoke-RestMethod ... > result.json` 在 PS 5.1 下会生成 UTF-16 LE 文件。如果你后续用 `cat`（WSL 环境）或 Python 默认参数 `open('result.json')` 读取，中文全灭。必须避免 `>`，改用 `Out-File -Encoding utf8`。

- **ConvertTo-Json 的深度问题**  
  默认序列化深度只有 2 层。复杂嵌套对象会被截断，导致数据丢失。务必使用 `-Depth` 参数，例如 `ConvertTo-Json -Depth 10`。

- **MCP 工具开发中的 stdout 编码**  
  如果你在 Windows 上用 PowerShell 写 MCP 工具（比如一个本地脚本 Server），并通过 stdin/stdout 传递 JSON，客户可能读到的仍是乱码。需要在脚本中执行：
  ```powershell
  [Console]::OutputEncoding = [System.Text.Encoding]::UTF8
  ```
  并确保返回的数据直接用 `Write-Output` 结合 `Out-File` 或管道时不发生二次编码转换。

- **VSCode/IDE 编码误判**  
  文件本身已是 UTF-8，但 IDE 以 GBK 方式重新打开，也会显示乱码。检查窗口右下角的编码标记，强制以 UTF-8 重新打开。

## 6. 可复用建议

1. **在 Agent 流水线中规定输出编码**：所有 PowerShell 脚本的输出文件统一用 `Out-File -Encoding utf8` 或 `Set-Content -Encoding utf8`，禁止使用 `>`。
2. **尽量使用 -OutFile**：对于纯粹的 API 响应转存，`Invoke-RestMethod -OutFile` 是最干净的方法。
3. **切换到 pwsh**：如果宿主环境允许，卸载 Windows PowerShell 的胡思乱想，拥抱 pwsh。
4. **编写 MCP 工具时增加编码守护**：在工具脚本的 `begin` 块中设置 `[Console]::OutputEncoding`，防御性编程。
5. **团队内部提供“防乱码”模板**：例如提供一个封装好的 `Invoke-ApiToUtf8` 函数，内部规范所有输出。

## 7. 总结

PowerShell 在 Windows 上的中文乱码问题，本质是历史相容性与 UTF-8 推广之间的冲突。它不是什么神秘 bug，而是三套默认编码（控制台、管道、文件输出）耦合作用的结果。理解了这一点，并且养成“输出即指定编码”的习惯，你的 Agent 和自动化流水线就能稳定吞吐中文 JSON。最后记住一句话：在所有自动化文本流中，显式优于默认，UTF-8 优于一切。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-16/1896689626bf5598.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-16/0f40e45e7704489f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-16/ea93305de5b733de.png)

