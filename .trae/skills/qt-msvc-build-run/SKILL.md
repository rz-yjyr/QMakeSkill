---
name: "qt-msvc-build-run"
description: "Builds and runs Qt/C++ projects using MSVC + qmake + jom. Invoke when user asks to build, compile, or run a Qt project, or needs a Qt build script template."
---

# Qt MSVC Build & Run

This skill provides a complete Windows batch script for building Qt/C++ projects with MSVC compiler, qmake, and jom.

## Features

- **One-step build**: Clean → Setup MSVC env → qmake → Compile → Run
- **Configurable paths**: Qt directory and MSVC vcvars path
- **Error handling**: Checks for required tools at each step
- **Auto-run**: Starts the application after successful build

## Prerequisites

- Qt installed (e.g., Qt 6.6.3 MSVC2019 64-bit)
- Microsoft Visual Studio with C++ build tools
- jom (comes with Qt Creator tools)

## Configuration

This build script supports two configuration modes:

### Mode 1: Auto-config (Recommended)

Run `init.bat` first (from the `qt-msvc-init` skill) to auto-detect and generate `build_config.bat`.

### Mode 2: Manual Config

Edit these variables at the top of the script:

```batch
set QT_DIR=D:\Qt\6.6.3\msvc2019_64
set VCVARS_PATH=D:\Program Files\Microsoft Visual Studio\2022\Community\VC\Auxiliary\Build\vcvars64.bat
set BUILD_DIR_NAME=build
```

## The Build Script

Save this as `build.bat` in your project root:

```batch
@echo off
chcp 65001 >nul
setlocal EnableDelayedExpansion

REM ========================================
REM Load Configuration
REM ========================================
if exist "%~dp0build_config.bat" (
    call "%~dp0build_config.bat"
    echo Configuration loaded from build_config.bat
    echo   Qt:      %QT_DIR%
    echo   MSVC:    %VCVARS_PATH%
    echo   Project: %PROJECT_NAME%
    echo.
) else (
    REM ========================================
    REM Manual Configuration (fallback)
    REM ========================================
    set QT_DIR=D:\Qt\6.6.3\msvc2019_64
    set VCVARS_PATH=D:\Program Files\Microsoft Visual Studio\2022\Community\VC\Auxiliary\Build\vcvars64.bat
    set BUILD_DIR_NAME=build
    set PROJECT_NAME=XPolMenuTest
    echo Using manual configuration
    echo.
)

REM ========================================
REM Internal Configuration
REM ========================================
set PROJECT_DIR=%~dp0
set BUILD_DIR=%PROJECT_DIR%%BUILD_DIR_NAME%
set QMAKE=%QT_DIR%\bin\qmake.exe
set JOM=%QT_DIR%\..\..\Tools\QtCreator\bin\jom\jom.exe
set EXE_PATH=%BUILD_DIR%\release\%PROJECT_NAME%.exe

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
    echo     Error: vcvars64.bat not found at %VCVARS_PATH%
    echo     Please run init.bat to configure your environment
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

"%QMAKE%" "%PROJECT_DIR%%PROJECT_NAME%.pro"
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
    echo     Error: exe not found at %EXE_PATH%
    exit /b 1
)

echo.
echo ========================================
echo Build and run completed successfully
echo ========================================

endlocal
```

## Usage

### First Time Setup (Recommended)

1. Place `init.bat` and `build.bat` in your Qt project root
2. Run `init.bat` once to auto-detect your environment
3. Run `build.bat` to build and run your project

### Manual Setup (Alternative)

1. Place `build.bat` in your Qt project root
2. Edit the manual configuration section in `build.bat`
3. Run `build.bat`

## Customization Tips

- **Change project name**: Detected automatically by `init.bat`, or update `PROJECT_NAME` manually
- **Debug build**: Change `release` to `debug` in `EXE_PATH`
- **Keep build directory**: Remove the clean step if you want incremental builds
- **Different Qt version**: Run `init.bat` again, or update `QT_DIR` manually

## Integration with qt-msvc-init

These two skills work together:

1. **qt-msvc-init**: Run once to detect and save configuration
2. **qt-msvc-build-run**: Reads the saved config and builds/runs the project

```
Project Root/
├── init.bat          (from qt-msvc-init skill)
├── build.bat         (from qt-msvc-build-run skill)
├── build_config.bat  (generated by init.bat - user-specific)
├── .gitignore        (should ignore build_config.bat)
└── YourProject.pro
```

## How to Share on GitHub

To share this skill with others via GitHub:

1. **Create a repository** on GitHub (e.g., `qt-msvc-build-skill`)
2. **Copy the `.trae/skills/qt-msvc-build-run/` folder** to your repository
3. **Add a README.md** explaining how to install the skill
4. **Users install it** by copying the folder to their project's `.trae/skills/` directory

### Repository Structure for GitHub

```
qt-msvc-build-skill/
├── README.md
└── qt-msvc-build-run/
    └── SKILL.md
```

### Example README.md for GitHub

```markdown
# Qt MSVC Build Skill for Trae IDE

A Trae IDE skill for building and running Qt/C++ projects on Windows with MSVC.

## Installation

1. Download or clone this repository
2. Copy the `qt-msvc-build-run` folder to your project's `.trae/skills/` directory:
   ```
   your-project/
   └── .trae/
       └── skills/
           └── qt-msvc-build-run/
               └── SKILL.md
   ```
3. Restart Trae IDE or reload the window

## Usage

Once installed, ask the AI assistant to "build the project" or "run the Qt application".

## Requirements

- Windows with MSVC build tools
- Qt (6.x recommended)
- jom (included with Qt Creator)
```
