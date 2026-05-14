---
name: "qmake-build"
description: "构建 Qt/C++ 项目并报告结果。检查当前目录下的 qmake-build.bat 并验证路径有效性，若不存在或路径失效则询问用户使用 qmake-init 初始化或修复。当用户输入 /qmake-build 或要求构建/编译项目时调用。"
---

# Build Skill

## 用途

执行构建脚本编译 Qt/C++ 项目并报告结果。优先检查项目目录下的 `qmake-build.bat`，验证其中 Qt/MSVC/jom 路径是否仍然有效。若脚本不存在或路径失效，询问用户是否使用 `qmake-init` 初始化或修复，不自动执行任何操作。

## 触发时机

- 用户输入 `/qmake-build`
- 用户要求构建、编译或运行项目
- 用户希望验证代码变更是否能正确编译
- 完成需要验证的代码修改后

## 执行步骤

### 步骤 1：检查 qmake-build.bat

```powershell
$hasBuildScript = Test-Path "qmake-build.bat"
```

- **若存在**，进入步骤 2（路径有效性检查）。
- **若不存在**，进入步骤 4（询问用户是否初始化）。

### 步骤 2：验证 qmake-build.bat 中的路径有效性

读取 `qmake-build.bat` 中的 `QT_DIR`、`VCVARS_PATH`、`JOM` 路径，检查是否仍然有效：

```powershell
$batContent = Get-Content "qmake-build.bat" -Raw
# 提取路径变量
$qtDir = [regex]::Match($batContent, 'set QT_DIR=(.+)').Groups[1].Value.Trim()
$vcvarsPath = [regex]::Match($batContent, 'set VCVARS_PATH=(.+)').Groups[1].Value.Trim()
$jomPath = [regex]::Match($batContent, 'set JOM=(.+)').Groups[1].Value.Trim()

# 检查有效性
$qtValid = Test-Path "$qtDir\bin\qmake.exe"
$vcvarsValid = Test-Path $vcvarsPath
$jomValid = Test-Path $jomPath
```

- **若所有路径均有效**，进入步骤 3（直接执行）。
- **若有路径失效**（如 Qt/MSVC 升级或移动位置），进入步骤 5（询问用户是否修复）。

### 步骤 3：执行 qmake-build.bat

路径验证通过，**必须在 `cmd.exe` 中调用**构建脚本（`.bat` 在 PowerShell 中执行可能导致变量展开和错误处理行为异常）：

```powershell
cmd.exe /c "qmake-build.bat"
```

执行后进入步骤 6（捕获输出并报告结果）。

### 步骤 4：qmake-build.bat 不存在 — 询问用户是否初始化

使用 `AskUserQuestion` 询问用户：

> 当前目录未找到 `qmake-build.bat`，是否使用 `qmake-init` 扫描系统并生成构建脚本？

- **用户选择「是」**：调用 `qmake-init` 技能扫描系统、收集候选路径、生成 `qmake-build.bat`，然后回到步骤 3 执行（注意步骤 3 必须使用 `cmd.exe /c` 调用）。
- **用户选择「否」**：跳过构建，向用户说明需要先有构建脚本才能继续。

### 步骤 5：路径失效 — 询问用户是否修复

提取出失效的路径，向用户报告：

> 检测到 `qmake-build.bat` 中的以下路径已失效：
> - `QT_DIR` = `D:\Qt\6.6.3\msvc2019_64` （qmake.exe 不存在）
> - `VCVARS_PATH` = `...` （文件不存在）
>
> 是否使用 `qmake-init` 重新扫描系统并修复路径？

使用 `AskUserQuestion` 询问：

- **用户选择「是」**：调用 `qmake-init` 技能重新扫描，更新 `qmake-build.bat` 中的路径，然后回到步骤 3 执行（注意步骤 3 必须使用 `cmd.exe /c` 调用）。
- **用户选择「否」**：跳过构建，提示用户手动检查路径。

### 步骤 6：捕获并分析输出

- 捕获脚本的标准输出和错误输出
- 记录 exit code

### 步骤 7：向用户报告结果

- **成功**：构建完成且应用程序已启动
- **失败**：显示具体的错误信息，并进入错误处理流程

## 内嵌备用脚本

若用户拒绝初始化且当前目录无 `qmake-build.bat`，作为最后的回退手段，将以下内嵌脚本写入临时文件后执行。

> ⚠️ **注意**：内嵌脚本使用硬编码路径，仅在已知环境匹配时有效。优先使用上述步骤 1~5。

```batch
@echo off
chcp 65001 >nul

REM ========================================
REM User Configuration (Fallback)
REM ========================================
set QT_DIR=D:\Qt\6.6.3\msvc2019_64
set VCVARS_PATH=D:\Program Files\Microsoft Visual Studio\2022\Community\VC\Auxiliary\Build\vcvars64.bat
set BUILD_DIR_NAME=build

REM ========================================
REM Internal Configuration
REM ========================================
set PROJECT_DIR=%CD%\
set BUILD_DIR=%PROJECT_DIR%%BUILD_DIR_NAME%
set QMAKE=%QT_DIR%\bin\qmake.exe
set JOM=%QT_DIR%\..\..\Tools\QtCreator\bin\jom\jom.exe

REM ========================================
REM Find .pro file
REM ========================================
set "PRO_FILE="
for %%f in (%PROJECT_DIR%*.pro) do if not defined PRO_FILE set "PRO_FILE=%%~nxf"
if not defined PRO_FILE (
    echo Error: No .pro file found in %PROJECT_DIR%
    exit /b 1
)
echo Found project: %PRO_FILE%

REM EXE_PATH 必须在 PRO_FILE 赋值后定义，否则批处理解析时展开为空
set EXE_PATH=%BUILD_DIR%\release\%PRO_FILE:.pro=%.exe

REM ========================================
REM Step 0: Kill running process
REM ========================================
echo [0/4] Killing running process...
taskkill /F /IM %PRO_FILE:.pro=%.exe >nul 2>&1
echo     OK
echo.

REM ========================================
REM Step 1: Clean
REM ========================================
echo [1/4] Cleaning...
if exist "%BUILD_DIR%" (
    cd /d "%BUILD_DIR%"
    if exist "Makefile" "%JOM%" clean >nul 2>&1
    cd /d "%PROJECT_DIR%"
    rmdir /s /q "%BUILD_DIR%"
)
echo     OK
echo.

REM ========================================
REM Step 2: Setup Environment
REM ========================================
echo [2/4] Setting up MSVC environment...
if not exist "%VCVARS_PATH%" (
    echo     Error: vcvars64.bat not found
    exit /b 1
)
call "%VCVARS_PATH%" >nul
echo     OK
echo.

REM ========================================
REM Step 3: Build
REM ========================================
echo [3/4] Building...
mkdir "%BUILD_DIR%" >nul 2>&1
cd /d "%BUILD_DIR%"

"%QMAKE%" "%PROJECT_DIR%%PRO_FILE%"
if errorlevel 1 (
    echo     Error: qmake failed
    exit /b 1
)

"%JOM%"
if errorlevel 1 (
    echo     Error: jom failed
    exit /b 1
)
echo     OK
echo.

REM ========================================
REM Step 4: Run
REM ========================================
echo [4/4] Starting application...
if exist "%EXE_PATH%" (
    start "" "%EXE_PATH%"
    echo     OK
) else (
    echo     Error: exe not found
    exit /b 1
)

echo.
echo ========================================
echo Build and run completed successfully
echo ========================================
```

---

## 错误处理

如果构建失败，按以下优先级处理：

### 1. 分析错误类型

| 错误关键词 | 可能原因 | 处理策略 |
|-----------|---------|---------|
| `vcvars64.bat not found` | MSVC 环境未找到 | 调用 `qmake-init` 重新扫描，或提示用户安装 MSVC |
| `qmake failed` | .pro 文件错误或 Qt 环境问题 | 检查 .pro 文件语法，或重新运行 `qmake-init` |
| `jom failed` / `nmake failed` | 编译错误（语法错误、头文件缺失等） | 从输出中提取具体错误行，向用户解释 |
| `exe not found` | 构建未生成可执行文件 | 检查构建目录中的实际输出文件名 |
| `No .pro file found` | 当前目录没有 Qt 项目文件 | 提示用户切换到正确的项目目录 |

### 2. 自动修复策略

根据错误类型尝试自动修复：

- **路径类错误**（vcvars64 / qmake / jom 未找到）：
  - 自动调用 `qmake-init` 重新扫描系统工具链
  - 重新生成 `qmake-build.bat`
  - 询问用户确认后再次构建

- **编译类错误**（语法错误、链接错误等）：
  - 提取前 5 条具体错误信息
  - 用中文向用户解释错误原因
  - 询问："构建失败，是否需要自动修复？"
  - 仅在用户确认后才尝试修复代码

### 3. 报告格式

构建失败后向用户报告：

```
❌ 构建失败

错误类型：[具体类型]

关键错误信息：
  1. [错误1]
  2. [错误2]
  ...

建议：
  [针对该错误的具体建议]

是否需要自动修复？（是/否）
```

---

## 备注

- **执行流程**：检查 `qmake-build.bat` → 验证路径有效性 → 若失效询问修复 → 若不存在询问初始化 → 内嵌回退脚本
- **调用方式限制**：`qmake-build.bat` 必须通过 `cmd.exe /c "qmake-build.bat"` 调用，严禁在 PowerShell 中直接 `& "qmake-build.bat"`，以避免变量展开和错误处理行为差异导致的问题
- 构建系统：qmake + jom
- 输出可执行文件：`<当前目录>\build\release\<工程名>.exe`（自动匹配 .pro 文件名）
- **严禁手动拆分执行 qmake 或 jom，必须通过完整脚本执行**
- 若项目目录已有 `qmake-build.bat`，可直接双击运行，不依赖 Kimi 调用
