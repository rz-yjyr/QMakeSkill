---
name: "qmake-init"
description: "自动扫描系统并定位 Qt、MSVC、qmake、jom 等编译工具路径。若发现多个候选路径，提示用户选择。供 qmake-build 或其他构建技能动态获取环境配置。"
---

# Qt 环境扫描技能

## 用途
自动扫描系统中 Qt 和 MSVC 编译工具的安装位置，动态发现构建所需的工具链路径。当存在多个候选安装时，提示用户进行选择，替代硬编码路径。

## 触发时机
- 构建前需要确认工具链位置时
- 硬编码路径失效（如 Qt/MSVC 升级或安装在其他磁盘）时
- 用户要求查找 Qt/MSVC 环境时

## 扫描逻辑

### 1. 扫描 Qt 与 qmake
检查以下常见位置的 `bin\qmake.exe`：
- `C:\Qt\*\msvc2019_64\bin\qmake.exe`
- `D:\Qt\*\msvc2019_64\bin\qmake.exe`
- `E:\Qt\*\msvc2019_64\bin\qmake.exe`
- 版本目录通配匹配（如 `6.6.3`、`6.7.*` 等）

### 2. 扫描 MSVC vcvars64.bat
检查以下常见位置：
- `C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Auxiliary\Build\vcvars64.bat`
- `C:\Program Files\Microsoft Visual Studio\2022\Enterprise\VC\Auxiliary\Build\vcvars64.bat`
- `C:\Program Files\Microsoft Visual Studio\2022\Professional\VC\Auxiliary\Build\vcvars64.bat`
- `C:\Program Files\Microsoft Visual Studio\2022\BuildTools\VC\Auxiliary\Build\vcvars64.bat`
- `C:\Program Files\Microsoft Visual Studio\2019\Community\VC\Auxiliary\Build\vcvars64.bat`
- 以及 `D:\Program Files\...` 等其他盘符

### 3. 扫描 jom
基于找到的 Qt 路径推导：
- `<Qt 安装根目录>\Tools\QtCreator\bin\jom\jom.exe`
- 若 Qt 路径为 `D:\Qt\6.6.3\msvc2019_64`，则 jom 通常在 `D:\Qt\Tools\QtCreator\bin\jom\jom.exe`

## 执行步骤

### 步骤 1：收集所有候选路径

使用 PowerShell 执行扫描，收集所有匹配项（不要只取第一个）：

```powershell
# 扫描所有 qmake
$qmakeList = @("C:\Qt", "D:\Qt", "E:\Qt") | ForEach-Object {
    if (Test-Path $_) {
        Get-ChildItem "$($_)\*\msvc2019_64\bin\qmake.exe" -ErrorAction SilentlyContinue | Select-Object -ExpandProperty FullName
    }
}

# 扫描所有 vcvars64.bat
$vcvarsList = @(
    "C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Auxiliary\Build\vcvars64.bat",
    "C:\Program Files\Microsoft Visual Studio\2022\Enterprise\VC\Auxiliary\Build\vcvars64.bat",
    "C:\Program Files\Microsoft Visual Studio\2022\Professional\VC\Auxiliary\Build\vcvars64.bat",
    "C:\Program Files\Microsoft Visual Studio\2022\BuildTools\VC\Auxiliary\Build\vcvars64.bat",
    "C:\Program Files\Microsoft Visual Studio\2019\Community\VC\Auxiliary\Build\vcvars64.bat",
    "D:\Program Files\Microsoft Visual Studio\2022\Community\VC\Auxiliary\Build\vcvars64.bat",
    "D:\Program Files\Microsoft Visual Studio\2022\Enterprise\VC\Auxiliary\Build\vcvars64.bat",
    "D:\Program Files\Microsoft Visual Studio\2022\Professional\VC\Auxiliary\Build\vcvars64.bat"
) | Where-Object { Test-Path $_ }

# 扫描所有 jom（基于 Qt 路径推导）
$jomList = @()
$qmakeList | ForEach-Object {
    $qtDir = Split-Path (Split-Path (Split-Path $_)) -Parent
    $qtRoot = Split-Path $qtDir
    $jomPath = Join-Path $qtRoot "Tools\QtCreator\bin\jom\jom.exe"
    if (Test-Path $jomPath) { $jomList += $jomPath }
    else {
        $qtRoot2 = Split-Path (Split-Path $qtDir)
        $jomPath2 = Join-Path $qtRoot2 "Tools\QtCreator\bin\jom\jom.exe"
        if (Test-Path $jomPath2) { $jomList += $jomPath2 }
    }
}
$jomList = $jomList | Select-Object -Unique

# 输出收集结果
Write-Host "=== qmake 候选 ==="
$qmakeList | ForEach-Object { Write-Host "  $_" }
Write-Host "=== vcvars64.bat 候选 ==="
$vcvarsList | ForEach-Object { Write-Host "  $_" }
Write-Host "=== jom 候选 ==="
$jomList | ForEach-Object { Write-Host "  $_" }
```

### 步骤 2：处理多候选情况

- **若某类工具只找到 1 个路径**：直接确认使用该路径。
- **若某类工具找到多个路径**：列出所有候选，**提示用户进行选择**。
- **若某类工具未找到**：报告未找到，并询问用户是否手动指定。

使用 AskUserQuestion 呈现选项（示例）：

```
发现多个 Qt 版本，请选择：
1. D:\Qt\6.6.3\msvc2019_64\bin\qmake.exe
2. D:\Qt\6.7.0\msvc2019_64\bin\qmake.exe
3. D:\Qt\6.5.3\msvc2019_64\bin\qmake.exe
```

同理处理 MSVC 和 jom。

### 步骤 3：输出最终配置

用户选择完成后，输出最终确定的 JSON 配置：

```json
{
  "QT_DIR": "D:\\Qt\\6.6.3\\msvc2019_64",
  "QMAKE": "D:\\Qt\\6.6.3\\msvc2019_64\\bin\\qmake.exe",
  "VCVARS_PATH": "D:\\Program Files\\Microsoft Visual Studio\\2022\\Community\\VC\\Auxiliary\\Build\\vcvars64.bat",
  "JOM": "D:\\Qt\\Tools\\QtCreator\\bin\\jom\\jom.exe"
}
```

## 输出格式

扫描完成后返回 JSON 格式路径配置：
```json
{
  "QT_DIR": "<用户选择的 Qt 目录>",
  "QMAKE": "<用户选择的 qmake 路径>",
  "VCVARS_PATH": "<用户选择的 vcvars64.bat 路径>",
  "JOM": "<用户选择的 jom 路径>"
}
```

## 备注
- 若某项未找到，对应字段返回 `null`，并提示用户手动输入
- 支持 Qt 6.x 和 Qt 5.x 的自动识别
- 用户选择的结果可保存供后续 `qmake-build` 技能使用
