---
name: "qt-msvc-init"
description: "Initializes Qt/MSVC build configuration by auto-detecting Qt and MSVC paths, or creating a user config file. Invoke when setting up a new Qt project or when build environment needs configuration."
---

# Qt MSVC Build Environment Initializer

This skill automatically detects and configures Qt and MSVC build environments for Windows projects.

## Features

- **Auto-detect Qt installation**: Scans common Qt installation paths
- **Auto-detect MSVC**: Finds Visual Studio vcvars64.bat
- **Generate config file**: Creates `build_config.bat` for the build script to read
- **Interactive fallback**: Asks user for paths when auto-detection fails

## Auto-Detection Logic

### Qt Detection (in order)

1. Environment variable: `QT_DIR`
2. Common paths:
   - `C:\Qt\*\msvc2019_64`
   - `C:\Qt\*\msvc2022_64`
   - `D:\Qt\*\msvc2019_64`
   - `D:\Qt\*\msvc2022_64`
   - `E:\Qt\*\msvc2019_64`
   - `E:\Qt\*\msvc2022_64`
3. User input (if all above fail)

### MSVC Detection (in order)

1. Environment variable: `VCVARS_PATH`
2. **Dynamic disk scan**: Searches all available drives (C, D, E, F, G...) for:
   - `Program Files\Microsoft Visual Studio\*\*\VC\Auxiliary\Build\vcvars64.bat`
   - `Program Files (x86)\Microsoft Visual Studio\*\*\VC\Auxiliary\Build\vcvars64.bat`
3. **Hardcoded fallback paths**:
   - `C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Auxiliary\Build\vcvars64.bat`
   - `C:\Program Files\Microsoft Visual Studio\2022\Professional\VC\Auxiliary\Build\vcvars64.bat`
   - `C:\Program Files\Microsoft Visual Studio\2022\Enterprise\VC\Auxiliary\Build\vcvars64.bat`
   - `C:\Program Files (x86)\Microsoft Visual Studio\2022\Community\VC\Auxiliary\Build\vcvars64.bat`
   - `C:\Program Files (x86)\Microsoft Visual Studio\2019\Community\VC\Auxiliary\Build\vcvars64.bat`
   - `D:\Program Files\Microsoft Visual Studio\2022\Community\VC\Auxiliary\Build\vcvars64.bat`
4. User input (if all above fail)

## The Init Script

Save this as `init.bat` in your project root:

```batch
@echo off
chcp 65001 >nul
setlocal EnableDelayedExpansion

echo ========================================
echo Qt/MSVC Build Environment Initializer
echo ========================================
echo.

set CONFIG_FILE=%~dp0build_config.bat

REM ========================================
REM Step 1: Detect Qt
REM ========================================
echo [1/3] Detecting Qt installation...

set QT_DIR=

REM Check environment variable first
if defined QT_DIR_ENV (
    if exist "%QT_DIR_ENV%\bin\qmake.exe" (
        set QT_DIR=%QT_DIR_ENV%
        echo     Found via QT_DIR environment variable: !QT_DIR!
        goto :qt_found
    )
)

REM Search common Qt paths
for %%D in (C D E) do (
    for %%V in (6.6.3 6.5.3 6.7.0 6.8.0) do (
        for %%A in (msvc2019_64 msvc2022_64) do (
            if exist "%%D:\Qt\%%V\%%A\bin\qmake.exe" (
                set QT_DIR=%%D:\Qt\%%V\%%A
                echo     Found Qt: !QT_DIR!
                goto :qt_found
            )
        )
    )
)

REM Try wildcard search
for %%D in (C D E) do (
    for /d %%Q in ("%%D:\Qt\*") do (
        for %%A in (msvc2019_64 msvc2022_64) do (
            if exist "%%Q\%%A\bin\qmake.exe" (
                set QT_DIR=%%Q\%%A
                echo     Found Qt: !QT_DIR!
                goto :qt_found
            )
        )
    )
)

:qt_not_found
echo     Could not auto-detect Qt installation.
set /p QT_DIR="Please enter Qt path (e.g., D:\Qt\6.6.3\msvc2019_64): "
if not exist "!QT_DIR!\bin\qmake.exe" (
    echo     Error: qmake.exe not found at !QT_DIR!\bin
    exit /b 1
)

:qt_found
echo     OK
echo.

REM ========================================
REM Step 2: Detect MSVC
REM ========================================
echo [2/3] Detecting MSVC environment...

set VCVARS_PATH=

REM Check environment variable first
if defined VCVARS_PATH_ENV (
    if exist "%VCVARS_PATH_ENV%" (
        set VCVARS_PATH=%VCVARS_PATH_ENV%
        echo     Found via VCVARS_PATH environment variable: !VCVARS_PATH!
        goto :msvc_found
    )
)

REM Dynamic disk scan for MSVC
for %%D in (C D E F G H I J K L M N O P Q R S T U V W X Y Z) do (
    if exist "%%D:\" (
        REM Search Program Files
        for /d %%Y in ("%%D:\Program Files\Microsoft Visual Studio\*") do (
            for /d %%E in ("%%Y\*") do (
                if exist "%%E\VC\Auxiliary\Build\vcvars64.bat" (
                    set VCVARS_PATH=%%E\VC\Auxiliary\Build\vcvars64.bat
                    echo     Found MSVC: !VCVARS_PATH!
                    goto :msvc_found
                )
            )
        )
        REM Search Program Files (x86)
        for /d %%Y in ("%%D:\Program Files (x86)\Microsoft Visual Studio\*") do (
            for /d %%E in ("%%Y\*") do (
                if exist "%%E\VC\Auxiliary\Build\vcvars64.bat" (
                    set VCVARS_PATH=%%E\VC\Auxiliary\Build\vcvars64.bat
                    echo     Found MSVC: !VCVARS_PATH!
                    goto :msvc_found
                )
            )
        )
    )
)

REM Fallback: Search common hardcoded paths
for %%P in (
    "C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Auxiliary\Build\vcvars64.bat"
    "C:\Program Files\Microsoft Visual Studio\2022\Professional\VC\Auxiliary\Build\vcvars64.bat"
    "C:\Program Files\Microsoft Visual Studio\2022\Enterprise\VC\Auxiliary\Build\vcvars64.bat"
    "C:\Program Files (x86)\Microsoft Visual Studio\2022\Community\VC\Auxiliary\Build\vcvars64.bat"
    "C:\Program Files (x86)\Microsoft Visual Studio\2022\Professional\VC\Auxiliary\Build\vcvars64.bat"
    "C:\Program Files (x86)\Microsoft Visual Studio\2022\Enterprise\VC\Auxiliary\Build\vcvars64.bat"
    "C:\Program Files (x86)\Microsoft Visual Studio\2019\Community\VC\Auxiliary\Build\vcvars64.bat"
    "C:\Program Files (x86)\Microsoft Visual Studio\2019\Professional\VC\Auxiliary\Build\vcvars64.bat"
    "C:\Program Files (x86)\Microsoft Visual Studio\2019\Enterprise\VC\Auxiliary\Build\vcvars64.bat"
    "D:\Program Files\Microsoft Visual Studio\2022\Community\VC\Auxiliary\Build\vcvars64.bat"
    "D:\Program Files\Microsoft Visual Studio\2022\Professional\VC\Auxiliary\Build\vcvars64.bat"
    "D:\Program Files\Microsoft Visual Studio\2022\Enterprise\VC\Auxiliary\Build\vcvars64.bat"
) do (
    if exist %%P (
        set VCVARS_PATH=%%P
        echo     Found MSVC: !VCVARS_PATH!
        goto :msvc_found
    )
)

:msvc_not_found
echo     Could not auto-detect MSVC installation.
set /p VCVARS_PATH="Please enter vcvars64.bat path: "
if not exist "!VCVARS_PATH!" (
    echo     Error: vcvars64.bat not found at !VCVARS_PATH!
    exit /b 1
)

:msvc_found
echo     OK
echo.

REM ========================================
REM Step 3: Detect Project Name
REM ========================================
echo [3/3] Detecting project name...

set PROJECT_NAME=
for %%F in ("%~dp0*.pro") do (
    set PROJECT_NAME=%%~nF
    echo     Found project: !PROJECT_NAME!
    goto :project_found
)

:project_not_found
echo     Could not auto-detect project name.
set /p PROJECT_NAME="Please enter project name (without .pro extension): "

:project_found
echo     OK
echo.

REM ========================================
REM Step 4: Generate Config File
REM ========================================
echo Generating configuration file...
(
echo @echo off
echo REM ========================================
echo REM Qt/MSVC Build Configuration
echo REM Generated by init.bat
echo REM ========================================
echo.
echo set QT_DIR=!QT_DIR!
echo set VCVARS_PATH=!VCVARS_PATH!
echo set PROJECT_NAME=!PROJECT_NAME!
echo set BUILD_DIR_NAME=build
echo set EXE_PATH=%%BUILD_DIR%%\release\!PROJECT_NAME!.exe
) > "%CONFIG_FILE%"

echo     Config saved to: %CONFIG_FILE%
echo.
echo ========================================
echo Initialization completed successfully
echo ========================================
echo.
echo Detected configuration:
echo   Qt:      !QT_DIR!
echo   MSVC:    !VCVARS_PATH!
echo   Project: !PROJECT_NAME!
echo.

endlocal
```

## Usage

1. Place `init.bat` in your Qt project root (same directory as `.pro` file)
2. Run `init.bat` once to generate `build_config.bat`
3. The build script will automatically read `build_config.bat`

## How It Works

```
User runs init.bat
    ↓
Auto-detects Qt path
    ↓
Auto-detects MSVC path
    ↓
Auto-detects project name (from .pro file)
    ↓
Generates build_config.bat
    ↓
Build script reads build_config.bat
```

## Integration with Build Script

The `qt-msvc-build-run` skill's `build.bat` should be modified to read the config file:

```batch
REM Load configuration if exists
if exist "%~dp0build_config.bat" (
    call "%~dp0build_config.bat"
) else (
    echo Error: build_config.bat not found. Please run init.bat first.
    exit /b 1
)
```

## Git Ignore

Add this to your `.gitignore` so user-specific configs aren't committed:

```gitignore
# Qt/MSVC build configuration (user-specific)
build_config.bat
```

## Environment Variables (Optional)

Users can set these environment variables to skip auto-detection:

- `QT_DIR_ENV`: Path to Qt (e.g., `D:\Qt\6.6.3\msvc2019_64`)
- `VCVARS_PATH_ENV`: Path to vcvars64.bat

## Sharing on GitHub

When sharing this skill:

1. Include `init.bat` in the repository
2. Document that users should run `init.bat` before first build
3. Add `build_config.bat` to `.gitignore` examples
4. Provide troubleshooting for common detection failures
