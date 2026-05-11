---
name: "qmake-build"
description: "使用内嵌的 build.bat 脚本构建 Qt/C++ 项目并报告结果。当用户输入 /qmake-build 或要求构建/编译项目时调用。"
---

# Build Skill

## 用途
执行内嵌的构建脚本编译 Qt/C++ 项目并报告结果。

## 触发时机
- 用户输入 `/qmake-build`
- 用户要求构建、编译或运行项目
- 用户希望验证代码变更是否能正确编译
- 完成需要验证的代码修改后

## 构建脚本（内嵌版）

以下是可直接内嵌执行的构建脚本（路径已固定，不依赖 `%~dp0`）：

```batch
@echo off
chcp 65001 >nul

REM ========================================
REM User Configuration
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

## 执行步骤

1. **检查外部脚本**：若当前目录下 `build.bat` 存在，直接执行：
   ```powershell
   & "build.bat"
   ```

2. **使用内嵌脚本（备用）**：若外部脚本不存在，将上述"内嵌版"脚本内容写入当前目录的 `build.bat` 后执行，或写入临时文件并在当前目录下执行。

3. 捕获并分析输出

4. 向用户报告结果：
   - **成功**：构建完成且应用程序已启动
   - **失败**：显示具体的错误信息，并询问是否需要自动修复

## 错误处理

如果构建失败：
1. 从输出中定位具体错误
2. 用中文向用户解释错误原因
3. 询问："构建失败，是否需要自动修复？"
4. 仅在用户确认后才进行修复

## 备注
- 构建工具：MSVC 2022 + Qt 6.6.3
- 构建系统：qmake + jom
- 输出可执行文件：`<当前目录>\build\release\<工程名>.exe`（自动匹配 .pro 文件名）
- **严禁手动拆分执行 qmake 或 jom，必须通过完整脚本执行**
