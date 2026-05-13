# Claude Code Token 消耗状态栏配置文档

## 功能概述

在 Claude Code 底部状态栏实时显示：
- **当前** — 当前会话累计 Token 消耗
- **今日** — 今日所有会话累计 Token 消耗
- **本周** — 本周所有会话累计 Token 消耗
- **Ctx** — 当前上下文窗口使用量（来自 Claude Code 内置数据）

数据来源：Claude Code 通过 stdin 传入的 `context_window` JSON（准确，非估算）

## 文件清单

将以下文件放到 `~/.claude/` 目录（Windows 为 `C:\Users\<用户名>\.claude\`）：

| 文件 | 用途 | 是否需手动创建 |
|------|------|-------------|
| `token-stats.ps1` | 状态栏脚本（PowerShell） | 是 |
| `token-stats.json` | 今日/本周累计数据 | 自动创建 |
| `token-30s-record.json` | 每30秒历史快照（最多3000条，按天清理） | 自动创建 |
| `settings.json` | Claude Code 配置 | 修改现有 |

## 1. 状态栏脚本

创建文件：`~/.claude/token-stats.ps1`

```powershell
# Claude Code Token 状态栏脚本（PowerShell 版）
$DB_PATH = Join-Path $env:USERPROFILE ".claude\token-stats.json"
$RECORD_PATH = Join-Path $env:USERPROFILE ".claude\token-30s-record.json"
$MAX_RECORDS = 3000

function Format-Tokens($num) {
    if ($num -ge 999950) { return "{0:N2}M" -f ($num / 1000000) }
    if ($num -ge 1000) { return "{0:N1}K" -f ($num / 1000) }
    return $num.ToString()
}

function Get-Today { return (Get-Date).ToString("yyyy-MM-dd") }
function Get-WeekStart {
    $d = Get-Date
    return $d.AddDays(-$d.DayOfWeek.value__).ToString("yyyy-MM-dd")
}
function Get-NowTime { return (Get-Date).ToString("HH:mm:ss") }

# 从 stdin 读取 JSON（Claude Code 传入）
$inputJson = $null
try {
    $stdinData = [System.Console]::In.ReadToEnd()
    if ($stdinData) { $inputJson = $stdinData | ConvertFrom-Json -AsHashtable }
} catch {}

# 加载统计
$stats = if (Test-Path $DB_PATH) {
    try { Get-Content $DB_PATH -Raw | ConvertFrom-Json -AsHashtable } catch { $null }
} else { $null }
if (-not $stats) { $stats = @{ sessions = @{}; daily = @{}; weekly = @{} } }
if (-not $stats.sessions) { $stats.sessions = @{} }
if (-not $stats.daily) { $stats.daily = @{} }
if (-not $stats.weekly) { $stats.weekly = @{} }

$today = Get-Today
$weekStart = Get-WeekStart

$sessionTotal = 0
$dailyTotal = 0
$weeklyTotal = 0
$ctxCurrent = 0
$ctxPercent = 0

if ($inputJson -and $inputJson.context_window) {
    $sessionId = if ($inputJson.session_id) { $inputJson.session_id } else { "default" }
    $cw = $inputJson.context_window

    $currentInput = if ($cw.total_input_tokens) { $cw.total_input_tokens } else { 0 }
    $currentOutput = if ($cw.total_output_tokens) { $cw.total_output_tokens } else { 0 }
    $sessionTotal = $currentInput + $currentOutput

    $ctxCurrent = $sessionTotal
    $ctxPercent = if ($cw.used_percentage) { $cw.used_percentage } else { 0 }

    # 初始化 session
    if (-not $stats.sessions[$sessionId]) {
        $stats.sessions[$sessionId] = @{ recordedInput = 0; recordedOutput = 0 }
    }
    $session = $stats.sessions[$sessionId]

    # 计算增量（避免重复累加）
    $inputDelta = [Math]::Max(0, $currentInput - $session.recordedInput)
    $outputDelta = [Math]::Max(0, $currentOutput - $session.recordedOutput)

    if ($inputDelta -gt 0 -or $outputDelta -gt 0) {
        if (-not $stats.daily[$today]) {
            $stats.daily[$today] = @{ input = 0; output = 0; total = 0 }
        }
        if (-not $stats.weekly[$weekStart]) {
            $stats.weekly[$weekStart] = @{ input = 0; output = 0; total = 0 }
        }

        $stats.daily[$today].input += $inputDelta
        $stats.daily[$today].output += $outputDelta
        $stats.daily[$today].total += $inputDelta + $outputDelta

        $stats.weekly[$weekStart].input += $inputDelta
        $stats.weekly[$weekStart].output += $outputDelta
        $stats.weekly[$weekStart].total += $inputDelta + $outputDelta

        $session.recordedInput = $currentInput
        $session.recordedOutput = $currentOutput

        $stats | ConvertTo-Json -Depth 10 | Set-Content $DB_PATH
    }

    # 记录30秒快照
    $records = if (Test-Path $RECORD_PATH) {
        try { Get-Content $RECORD_PATH -Raw | ConvertFrom-Json -AsHashtable } catch { $null }
    } else { $null }
    if (-not $records) { $records = @{ date = $today; records = @() } }
    if (-not $records.date) { $records.date = $today }
    if (-not $records.records) { $records.records = @() }

    # 按天清理
    if ($records.date -ne $today) {
        $records.date = $today
        $records.records = @()
    }

    $dailyVal = if ($stats.daily[$today]) { $stats.daily[$today].total } else { 0 }
    $weeklyVal = if ($stats.weekly[$weekStart]) { $stats.weekly[$weekStart].total } else { 0 }

    $records.records += @{
        time = Get-NowTime
        sessionTotal = $sessionTotal
        ctxPercent = $ctxPercent
        dailyTotal = $dailyVal
        weeklyTotal = $weeklyVal
    }

    # 限制最大条数（3000条，约25小时）
    if ($records.records.Count -gt $MAX_RECORDS) {
        $records.records = $records.records[-$MAX_RECORDS..-1]
    }

    $records | ConvertTo-Json -Depth 10 | Set-Content $RECORD_PATH
}

$dailyTotal = if ($stats.daily[$today]) { $stats.daily[$today].total } else { 0 }
$weeklyTotal = if ($stats.weekly[$weekStart]) { $stats.weekly[$weekStart].total } else { 0 }

# ANSI 颜色
$ESC = [char]27
$C_CYAN = "$ESC[36m"
$C_WHITE = "$ESC[37m"
$C_GRAY = "$ESC[90m"
$C_GREEN = "$ESC[32m"
$C_YELLOW = "$ESC[33m"
$C_RED = "$ESC[31m"
$C_RESET = "$ESC[0m"

$ctxColor = if ($ctxPercent -ge 80) { $C_RED } elseif ($ctxPercent -ge 50) { $C_YELLOW } else { $C_GREEN }

$output = "{0}Token{1}  {2}当前{3}{4}{5}  {6}|{7}  {8}今日{9}{10}{11}  {12}|{13}  {14}本周{15}{16}{17}    {18}Ctx{19}  {20}{21}{22}{23}({24}%{25})" -f `
    $C_CYAN, $C_RESET, $C_GRAY, $C_RESET, $C_WHITE, (Format-Tokens $sessionTotal), `
    $C_GRAY, $C_RESET, $C_GRAY, $C_RESET, $C_WHITE, (Format-Tokens $dailyTotal), `
    $C_GRAY, $C_RESET, $C_GRAY, $C_RESET, $C_WHITE, (Format-Tokens $weeklyTotal), `
    $C_CYAN, $C_RESET, $ctxColor, (Format-Tokens $ctxCurrent), $C_RESET, $C_GRAY, $ctxPercent, $C_RESET

Write-Output $output
```

## 2. Claude Code 配置

修改文件：`~/.claude/settings.json`，添加 `statusLine` 配置：

```json
{
  "statusLine": {
    "type": "command",
    "command": "pwsh.exe -NoProfile -ExecutionPolicy Bypass -File \"C:/Users/<用户名>/.claude/token-stats.ps1\"",
    "refreshInterval": 30
  }
}
```

**注意**：`<用户名>` 替换为实际用户名。Windows 支持正斜杠路径，无需转义反斜杠。

## 3. 使用步骤

1. **安装 PowerShell 7+**（`pwsh.exe`）
2. **复制脚本**：将 `token-stats.ps1` 放到 `~/.claude/`
3. **修改配置**：在 `settings.json` 中添加 `statusLine`
4. **重启 Claude Code**

## 数据文件格式

### token-stats.json（累计统计）

```json
{
  "sessions": {
    "session_xxx": {
      "recordedInput": 120000,
      "recordedOutput": 50000
    }
  },
  "daily": {
    "2026-05-13": {
      "input": 120000,
      "output": 50000,
      "total": 170000
    }
  },
  "weekly": {
  "2026-05-10": {
      "input": 120000,
      "output": 50000,
      "total": 170000
    }
  }
}
```

### token-30s-record.json（30秒快照）

```json
{
  "date": "2026-05-13",
  "records": [
    {
      "time": "14:30:00",
      "sessionTotal": 120000,
      "ctxPercent": 60,
      "dailyTotal": 150000,
      "weeklyTotal": 200000
    }
  ]
}
```

**清理策略**：
- `token-30s-record.json`：按天清理（非今天数据全部清空），最多保留 3000 条
- `token-stats.json`：daily 保留 7 天，weekly 保留 30 天

## 原理说明

1. Claude Code 每 30 秒执行一次状态栏脚本
2. 通过 stdin 传入 JSON 数据（包含 `context_window` 字段）
3. 脚本提取当前会话的累计 token 数
4. 计算增量（当前值 - 上次记录值），累加到今日/本周
5. 输出格式化后的状态栏字符串

**无需代理**：直接读取 Claude Code 内置数据，准确度高，无性能开销。
