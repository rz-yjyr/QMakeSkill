# Qt MSVC Build Skills for Kimi Code CLI

一套用于 [Kimi Code CLI](https://github.com/BinNong/kimi-code-cli) 的 Qt/MSVC 构建技能，支持自动检测开发环境、一键构建运行 Qt 项目，以及构建失败时的智能诊断修复。

## 包含的技能

| 技能 | 功能 | 文件 |
|------|------|------|
| `qmake-init` | 自动扫描 Qt、MSVC、qmake、jom 等工具路径，生成配置 | `qmake-init/SKILL.md` |
| `qmake-build` | 执行清理→构建→运行，支持内嵌脚本与外部 `build.bat` | `qmake-build/SKILL.md` |
| `qmake-diagnose` | 构建失败时分析错误日志，匹配已知问题并提供修复方案 | `qmake-diagnose/SKILL.md` |

## 快速开始

### 1. 安装技能

将本仓库复制到 Kimi Code CLI 的 skills 目录：

```powershell
# 默认 skills 目录路径示例
C:\Users\<用户名>\.agents\skills\
```

安装后的目录结构：

```
skills/
├── qmake-init/
│   └── SKILL.md      # 环境扫描技能
├── qmake-build/
│   └── SKILL.md      # 构建技能
└── qmake-diagnose/
    └── SKILL.md      # 故障诊断技能
```

在 Qt 项目目录中，Kimi Code CLI 会自动识别并调用这些技能。

### 2. 初始化环境（首次使用）

在 Kimi Code CLI 中输入：

```
/qmake-init
```

或询问：

> "初始化 Qt 构建环境"

`qmake-init` 会：
- 扫描常见路径（`C:\Qt`、`D:\Qt`、`E:\Qt` 等）查找 Qt 安装
- 扫描 `Program Files\Microsoft Visual Studio` 查找 MSVC 环境
- 基于 Qt 路径推导 jom 位置
- 若发现多个候选，提示用户选择
- 输出 JSON 格式的最终配置

### 3. 构建并运行项目

在 Kimi Code CLI 中输入：

```
/qmake-build
```

或询问：

> "构建并运行项目"

`qmake-build` 会：
- 检查当前目录下的 `build.bat`，若存在则直接执行
- 若不存在，使用内嵌脚本执行：
  - 终止正在运行的进程
  - 清理旧构建目录
  - 设置 MSVC 环境（`vcvars64.bat`）
  - 执行 `qmake` + `jom` 编译
  - 自动启动生成的可执行文件

### 4. 构建失败时诊断

如果构建失败，`qmake-diagnose` 会自动或按需调用：

```
/qmake-diagnose
```

它会分析构建日志，匹配 9 类常见错误（如 `QMAKE_PROJECT_DEPTH` 配置错误、变量展开顺序错误、缺少 Qt 模块、MOC/UIC 错误、LNK 链接错误等），并提供：
- **自动修复**：直接修改 `.pro` 或 `build.bat`
- **修复建议**：说明根因和方案，等待用户确认
- **排查引导**：对于未知错误，提供详细分析和建议

## 项目结构示例

```
your-project/
├── qmake-build/
│   └── SKILL.md           # 构建技能（从本仓库复制）
├── qmake-diagnose/
│   └── SKILL.md           # 诊断技能（从本仓库复制）
├── qmake-init/
│   └── SKILL.md           # 初始化技能（从本仓库复制）
├── build.bat              # 生成的构建脚本（可选，用户特定）
├── build_config.bat       # 生成的配置文件（用户特定，应加入 .gitignore）
├── .gitignore
└── YourProject.pro
```

## .gitignore 建议

```gitignore
# Qt/MSVC 构建配置（用户特定，不应提交）
build_config.bat

# 构建输出
build/
*.exe
*.obj
*.pdb
```

## 系统要求

- Windows 10/11
- Qt 5.x 或 6.x（MSVC 版本）
- Microsoft Visual Studio 2019/2022
- jom（随 Qt Creator 安装）

## 故障排除速查

| 问题 | 解决方案 |
|------|---------|
| 找不到 Qt | 运行 `/qmake-init` 重新扫描，或设置 `QT_DIR_ENV` 环境变量 |
| 找不到 MSVC | 运行 `/qmake-init` 重新扫描，或设置 `VCVARS_PATH_ENV` 环境变量 |
| `Error: dependent ... does not exist` | 检查 `.pro` 中是否有 `QMAKE_PROJECT_DEPTH = 0`，尝试删除或添加该项 |
| `Error: exe not found` 但文件已生成 | 检查 `build.bat` 中 `EXE_PATH` 是否定义在 `PRO_FILE` 赋值之后 |
| `fatal error C1083` | 缺少头文件或 Qt 模块，运行 `/qmake-diagnose` 自动分析 |
| `error LNK2019` | 缺少库依赖或实现，检查 `.pro` 的 `LIBS` 和源文件完整性 |
| `cl` 不是内部或外部命令 | MSVC 环境未正确加载，检查 `vcvars64.bat` 路径 |

## License

MIT License - 自由使用和修改。
