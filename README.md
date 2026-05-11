# Qt MSVC Build Skills for Trae IDE

一套用于 Trae IDE 的 Qt/MSVC 构建技能，支持自动检测开发环境并一键构建运行 Qt 项目。

## 包含的技能

| 技能 | 功能 | 文件 |
|------|------|------|
| `qt-msvc-init` | 自动检测 Qt 和 MSVC 路径，生成配置文件 | `.trae/skills/qt-msvc-init/SKILL.md` |
| `qt-msvc-build-run` | 读取配置并执行清理→构建→运行 | `.trae/skills/qt-msvc-build-run/SKILL.md` |

## 快速开始

### 1. 安装技能

将本仓库的 `.trae/skills/` 目录复制到你的 Qt 项目根目录：

```
your-project/
├── .trae/
│   └── skills/
│       ├── qt-msvc-init/
│       │   └── SKILL.md
│       └── qt-msvc-build-run/
│           └── SKILL.md
└── YourProject.pro
```

### 2. 初始化配置（首次使用）

在 Trae IDE 中询问 AI：

> "初始化 Qt 构建环境"

或手动创建 `init.bat`（内容见 `qt-msvc-init/SKILL.md`），然后运行：

```batch
init.bat
```

`init.bat` 会：
- 自动检测 Qt 安装路径（支持 C/D/E 盘常见位置）
- 自动检测 MSVC 环境（全盘扫描 + 常见路径）
- 自动检测项目名称（从 `.pro` 文件）
- 生成 `build_config.bat` 配置文件

### 3. 构建并运行项目

在 Trae IDE 中询问 AI：

> "构建并运行项目"

或手动创建 `build.bat`（内容见 `qt-msvc-build-run/SKILL.md`），然后运行：

```batch
build.bat
```

`build.bat` 会：
- 读取 `build_config.bat` 配置
- 清理旧构建目录
- 设置 MSVC 环境
- 执行 qmake + jom 编译
- 自动运行生成的可执行文件

## 项目结构示例

```
your-project/
├── .trae/
│   └── skills/
│       ├── qt-msvc-init/
│       │   └── SKILL.md      # 初始化技能
│       └── qt-msvc-build-run/
│           └── SKILL.md      # 构建技能
├── init.bat                   # 从 SKILL.md 提取的初始化脚本
├── build.bat                  # 从 SKILL.md 提取的构建脚本
├── build_config.bat           # 生成的配置文件（用户特定，应加入 .gitignore）
├── .gitignore
└── YourProject.pro
```

## .gitignore 建议

```gitignore
# Qt/MSVC 构建配置（用户特定，不应提交）
build_config.bat

# 构建输出
build/
```

## 环境变量（可选）

设置以下环境变量可跳过自动检测：

| 变量 | 说明 | 示例 |
|------|------|------|
| `QT_DIR_ENV` | Qt 安装路径 | `D:\Qt\6.6.3\msvc2019_64` |
| `VCVARS_PATH_ENV` | vcvars64.bat 路径 | `C:\Program Files\...\vcvars64.bat` |

## 自动检测逻辑

### Qt 检测顺序

1. 环境变量 `QT_DIR_ENV`
2. 常见路径：`C:\Qt\*\msvc2019_64`、`D:\Qt\*\msvc2022_64` 等
3. 通配符搜索：`\Qt\*\msvc*_64`
4. 交互式输入

### MSVC 检测顺序

1. 环境变量 `VCVARS_PATH_ENV`
2. **动态全盘扫描**：遍历所有可用磁盘，搜索 `Program Files\Microsoft Visual Studio\*\*\VC\Auxiliary\Build\vcvars64.bat`
3. **硬编码回退路径**：常见 VS2019/VS2022 安装位置
4. 交互式输入

## 系统要求

- Windows 10/11
- Qt 5.x 或 6.x（MSVC 版本）
- Microsoft Visual Studio 2019/2022
- jom（随 Qt Creator 安装）

## 手动配置（不使用 init.bat）

如果你不想使用自动检测，可以直接编辑 `build.bat` 中的手动配置部分：

```batch
set QT_DIR=D:\Qt\6.6.3\msvc2019_64
set VCVARS_PATH=D:\Program Files\Microsoft Visual Studio\2022\Community\VC\Auxiliary\Build\vcvars64.bat
set PROJECT_NAME=YourProject
```

## 故障排除

| 问题 | 解决方案 |
|------|---------|
| 找不到 Qt | 设置 `QT_DIR_ENV` 环境变量 |
| 找不到 MSVC | 设置 `VCVARS_PATH_ENV` 环境变量 |
| 找不到 .pro 文件 | 确保 `init.bat` 和 `.pro` 文件在同一目录 |
| 构建失败 | 检查 Qt 和 MSVC 版本是否匹配（如 msvc2019_64） |

## 分享与安装

### 分享给他人

1. 将本仓库推送到 GitHub
2. 他人克隆/下载后，复制 `.trae/skills/` 到他们的项目
3. 运行 `init.bat` 生成他们自己的配置

### 作为项目的一部分

将 `.trae/skills/` 目录加入版本控制（不包含 `build_config.bat`），这样团队成员可以直接使用。

## License

MIT License - 自由使用和修改。
