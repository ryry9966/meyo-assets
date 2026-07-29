---
title: Windows 中文 JSON API 调用：为什么 PowerShell 会把中文打坏
feedId: 30955
source: 综合讨论
publishedAt: 2026-07-30
---

## 背景
在 Agent、MCP 工具链与各类自动化插件实践中，Windows 环境经常需要用 PowerShell 充当“胶水”：调用大模型 JSON API，拿到中文响应，再喂给下一个步骤。每次看似简单的 `Invoke-RestMethod`，很容易在引入中文字符后，悄悄地把内容打成一串问号、方块或乱码。这个问题的根源并不在 JSON 本身，而在于 PowerShell 在不同版本与不同输出目标之间，对字符编码的处理存在多条不一致的路径。

## 问题拆解
API 返回的 HTTP 响应体通常是 UTF-8 编码的 JSON。`Invoke-RestMethod`（以及 `Invoke-WebRequest`）在内部正确地将字节流解码为 .NET 字符串，中文字符此时仍然完好。

真正的损坏发生在以下三个时刻：
1. **字符串被发送到控制台窗口**：PowerShell 5.1 控制台默认使用 Windows-1252 或系统 OEM 代码页，而非 UTF-8。如果 `[Console]::OutputEncoding` 不是 UTF-8，Write-Host 或直接输出中文时，部分字符可能无法映射，显示为 `?` 或乱码。
2. **通过管道写入文件或重定向**：`>>`、`>` 以及 `Out-File` 在 PowerShell 5.1 中默认采用 UTF-16 LE 编码。如果将 UTF-8 来源的中文直接写入文件，再用其他工具以 UTF-8 读取，就会看到乱码。更隐蔽的是，如果设置了 `$OutputEncoding`，它会影响 `>` 重定向时的编码转换，但许多脚本并没有显式设置它。
3. **与外部命令行工具交互**：例如将包含中文的字符串通过管道传给 `curl.exe` 或 `jq.exe`，PowerShell 会按照 `$OutputEncoding` 将字符串转换回字节流。若该编码不是 UTF-8，外部工具收到的就是已经损坏的数据。

典型的受损场景：用 `$response = Invoke-RestMethod ...` 拿到一个含中文 `message` 属性的对象，接着执行 `$response.message > result.txt`。看似优雅，但 `result.txt` 里中文已经阵亡。

## 复现步骤（最小化示例）
在 Windows 10/11 上，使用 PowerShell 5.1 控制台（非 ISE）执行：

```powershell
# 模拟一个返回中文 JSON 的 API（或直接用本地 HTTP 测试）
$json = '{"code":0,"msg":"操作成功，你好世界"}'
$response = $json | ConvertFrom-Json
$response.msg          # 控制台可能正常显示
$response.msg > test.txt
Get-Content test.txt   # 显示乱码或 ???
```

再用 NotePad++ 以 UTF-8 或 UTF-16 打开 test.txt，会发现内容并非期望的中文。原因在于 `>` 重定向使用了 PowerShell 的默认编码（通常为 UTF-16 LE），而很多外部工具预期 UTF-8。

另一个常见误伤是 `ConvertTo-Json` 时，中文变成 `\uXXXX` 转义序列。PowerShell 5.1 的 `ConvertTo-Json` 默认将所有非 ASCII 字符转义，虽然数据语义正确，但人眼不可读，也不利于在日志中直接 grep。这并非“打坏”，但容易与编码问题混淆。

## 做法/步骤（工程化修复）
### 1. 定位环境编码
先确认当前编码状态：
```powershell
[Console]::OutputEncoding.EncodingName   # 通常为 Windows-1252 或 OEM
$OutputEncoding.EncodingName            # 影响管道和重定向的外部命令编码
```
### 2. 统一使用 UTF-8 输出（推荐方案）
**首选：切换到 PowerShell 7（pwsh.exe）**。PowerShell 7 在多数场景下已经默认使用 UTF-8 编码，且 `$OutputEncoding` 自动设为 UTF-8 Without BOM，直接消除大部分编码裂隙。

如果必须停留在 PowerShell 5.1，则在脚本最顶部显式设置：
```powershell
$OutputEncoding = [System.Text.UTF8Encoding]::new($false)   # Without BOM
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
```
注意：`[Console]::OutputEncoding` 的设置只对当前控制台窗口生效，且可能受字体影响，某些旧字体无法渲染中文，此时仍显示方框，但实际编码无误。

### 3. 安全写入文件
永远不要依赖 `>` 或 `Out-File` 的默认编码。使用 `Set-Content` 或 `Out-File` 时显式指定 `-Encoding utf8`（PowerShell 5.1 中为 `utf8`，代表带 BOM 的 UTF-8；若需无 BOM，可用 `utf8NoBOM` 在 PowerShell 7 中，或使用 .NET 方法写入）。

```powershell
$response.msg | Out-File -FilePath result.txt -Encoding utf8
# 对于纯文本日志，更推荐使用 Add-Content，同样加 -Encoding utf8
```

如果需要生成可供其他服务消费的 JSON 文件，并且希望保留可读中文（不转义），在 PowerShell 7 中 `ConvertTo-Json` 默认不转义，在 5.1 中可借助辅助函数或直接使用 `System.Text.Json`。

### 4. 与外部程序交互
当把中文传递给 `curl`、`python` 等外部程序时，务必保证 `$OutputEncoding` 为 UTF-8。此外，通过 `Start-Process` 或管道传入时，可以使用 `[System.Text.Encoding]::UTF8.GetBytes()` 预先转换，再用 `-InputObject` 传递字节流，避免 PowerShell 二次解码破坏数据。

## 踩坑点速记
- **ISE 的障眼法**：PowerShell ISE 使用 WPF 文本框，有时中文显示正常，但一旦脚本在无人值守的 Task Scheduler 中以 SYSTEM 账户运行，输出重定向就暴露出编码问题。永远不要在 ISE 里验证编码行为。
- **`chcp 65001` 并不全面**：在 cmd 中设置 UTF-8 代码页无法完全控制 PowerShell 内的 `$OutputEncoding`，仅对部分外部命令有效，容易造成“有时候好了”的假象。
- **BOM 引发的互操作事故**：带 BOM 的 UTF-8 文件在某些 MCP 服务器或 JSON 解析器（如 Python 的 `json` 模块）中会被拒绝或产生隐形字符。如果下游工具对此敏感，必须生成无 BOM 的 UTF-8。
- **`Invoke-WebRequest` 的 `RawContentStream`**：如果手动处理字节流，记得用 `[System.Text.Encoding]::UTF8.GetString()` 解码，避免混合使用 `$response.Content`（它已被解码成字符串，可能已被破坏）。

## 可复用建议
- **脚本模板头部**：新建 PowerShell 自动化脚本时，直接复制下面两行：
  ```powershell
  $OutputEncoding = [System.Text.UTF8Encoding]::new($false)
  [Console]::OutputEncoding = [System.Text.Encoding]::UTF8
  ```
- **文件操作统一命令**：使用 `Set-Content -Encoding utf8` 和 `Add-Content -Encoding utf8` 替换 `>` 和 `>>`。
- **容器化保底**：如果无法控制 Windows 宿主机环境，将整个 Agent 逻辑封装在 Docker 容器（基于 `mcr.microsoft.com/powershell:lts-7.2` 或 Alpine+PowerShell），可以一劳永逸地避开 Windows 控制台编码问题。
- **回归测试**：在 CI 流程中加入包含中文的集成测试，用无 BOM UTF-8 读取输出文件并比对哈希，避免后续改动引入编码回退。

## 总结
Windows 上的 PowerShell 中文乱码，并非 JSON 解析的 bug，而是 PowerShell 在字符串输出、重定向与外部程序通信时，编码隐式协商失败。核心矛盾在于：API 返回的 UTF-8 数据，在被 PowerShell 处理成 .NET 字符串后，必须经过一层正确的编码重新输出，但传统 Windows 环境默认的代码页并不是 UTF-8。

解决方案并不复杂：**迁移到 PowerShell 7 + 显式指定文件编码 + 设置 `$OutputEncoding`**。这三个动作组合，可以将中文 JSON API 调用中的“坏字符”彻底踢出你的 Agent 流水线。对于长期维护的自动化项目，强烈建议将这些编码设置固化为团队代码规范，避免“本地跑得好，服务器全乱码”的经典回放。

---

## 配图

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-30/d0af53a6975833d2.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-30/acbae52372db4bc8.png)

