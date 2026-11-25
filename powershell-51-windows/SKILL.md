---
name: powershell-51-windows
description: Windows 11 / PowerShell 5.1 环境下的命令与脚本速查。只要涉及在这台机器上执行或生成 shell 命令就应当查阅：给出可直接运行的 PowerShell 命令、编写或审查 .ps1 与 .bat 脚本、通过终端面板执行命令、把 Unix 命令改写成 PowerShell、判断该用 Git Bash 还是 PowerShell，以及排查"命令在 Windows 上跑不起来"、中文乱码、执行策略被拦截等问题。即使用户没有提到 PowerShell，只要目标是在 Windows 上操作命令行，也先读本技能再动手。
---

# PowerShell 5.1 速查（Windows 11）

## 1. 先判定命令跑在哪

生成或改写命令前，先确认它会从哪个入口执行：

| 入口 | 实际 shell | 语法 |
|---|---|---|
| Bash 工具 | Git Bash | POSIX sh，路径 `/c/Users/...`，中文路径可用 |
| 代码块 / 终端面板 | Windows PowerShell 5.1 | 本文规则 |
| `pwsh` | 仅当确已安装才是 7 | 不要假设存在，先 `Get-Command pwsh` |

同一任务里不要混用：Bash 工具里写 `Get-ChildItem` 会失败，PowerShell 里写 `ls -la` 也会失败。

## 2. 终端面板的硬限制

终端面板在工具层就拒绝以下内容，不是 shell 的限制，换写法也绕不过去：管道 `|`、变量 `$x`、`$( )`、`@( )`、脚本块、`if`/`foreach`、反引号、注释、多行、非 ASCII 字符。

- 超出一条就写成 `.ps1` 文件，再执行 `powershell -ExecutionPolicy Bypass -File .\x.ps1`
- 路径含中文（如桌面目录）无法经终端面板传递，改走 Bash 工具

## 3. 5.1 没有的语法（直接报错）

- `a && b` → `a; if ($?) { b }`
- `a || b` → `a; if (-not $?) { b }`
- `$x ?? $y` → `if ($null -eq $x) { $y } else { $x }`
- `$c ? $a : $b` → `if ($c) { $a } else { $b }`
- `ForEach-Object -Parallel` → 普通 `foreach` 循环
- `ConvertFrom-Json -AsHashtable` → 手工转成哈希表
- `Out-File -Encoding utf8NoBOM` → 见第 6 节

## 4. 别名陷阱

- `curl`、`wget` 是 `Invoke-WebRequest` 的别名，只认 `-Uri` 这类参数，抄来的 `curl -X POST -d` 会失败；要真 curl 用 `curl.exe`
- `sc` 是 `Set-Content` 的别名；服务控制用 `sc.exe`
- `ls` `cat` `rm` `cp` `mv` `mkdir` 虽存在，但不接受 Unix 选项：`ls -la`、`rm -rf` 都会报错
- `rm -rf` → `Remove-Item -Recurse -Force`
- 没有 `grep`/`sed`/`awk`/`head`/`tail`/`uniq`：用 `Select-String`、`Select-Object -First/-Last`
- `find . -name` → `Get-ChildItem -Recurse -Filter`

## 5. 路径与引号

- 调用带空格的路径：`& "C:\Program Files\App\app.exe" 参数`（不加 `&` 会被当成字符串）
- `\` 不是转义符，PowerShell 的转义符是反引号
- 双引号内 `$` 会展开，单引号内不展开
- 原生程序的 `-` 开头参数被误解析时，用 `--%` 停止解析

## 6. 编码（5.1 最容易踩）

- `>` 重定向默认写 UTF-16LE（5.1 的 Out-File 默认 Unicode），下游按 UTF-8 读会乱码；要 UTF-8 用 `| Out-File -Encoding utf8`（带 BOM）
- 写无 BOM 的 UTF-8：`[IO.File]::WriteAllText($p, $s, (New-Object Text.UTF8Encoding $false))`
- 读 UTF-8 文本：`Get-Content -Encoding UTF8`
- 控制台中文乱码：`[Console]::OutputEncoding = [Text.Encoding]::UTF8`
- 传给原生程序时：`$OutputEncoding = [Text.Encoding]::UTF8`

## 7. 执行与退出码

- 默认执行策略 Restricted，脚本运行不了：加 `-ExecutionPolicy Bypass`
- 原生程序退出码看 `$LASTEXITCODE`，cmdlet 成败看 `$?`
- 与 `$null` 比较时把 `$null` 放在左侧：`$null -eq $x`。写成 `$x -eq $null` 时，若 `$x` 是数组会返回筛出的元素而不是布尔值

## 8. 常用对照

- `command -v` → `Get-Command`
- `echo $VAR` → `$env:VAR` 或 `Get-ChildItem Env:VAR`
- `sleep` → `Start-Sleep -Seconds`
- `2>&1` 可直接使用

## 9. cmd 与批处理

PowerShell 与 cmd 只在外部程序层通用，内置命令与批处理语法不兼容。

- 需要 cmd 语义时走逃生口：`cmd /c "dir /a /s"`、`cmd /c .\x.bat`
- cmd 的 `/a` `/s` `/f` `/q` 开关在 PowerShell 里无效，换成 `-Force` `-Recurse` 等参数
- cmd 语法一律不认：`%VAR%` → `$env:VAR`，`%~dp0` → `$PSScriptRoot`，`if errorlevel 1` → `$LASTEXITCODE -ge 1`，`goto` 与 `for %%i` 无对应
- `>nul 2>&1` → `*> $null`
- `&&` 在 cmd 可用，在 PowerShell 5.1 不可用
- `&` 在 cmd 是命令分隔符，在 PowerShell 是调用运算符
- 写 .bat 时：`@echo off`、`setlocal`、`call :label`，变量延迟展开用 `setlocal enabledelayedexpansion` + `!VAR!`
- 执行策略只作用于 PowerShell 脚本：`.bat` 不受限，`.ps1` 受
- `.bat` 里处理 UTF-8 前先 `chcp 65001 >nul`
