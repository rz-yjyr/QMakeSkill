---
name: "qmake-diagnose"
description: "Qt/C++ 构建故障诊断与修复向导。当 qmake-build 失败时优先调用，分析错误日志并提供针对性的解决方案或自动修复。"
---

# Qt 构建诊断技能

## 用途
在 `qmake-build` 构建失败时，**优先调用本技能**分析错误输出，匹配已知问题模式，提供解决方案或执行自动修复。

## 触发时机
- `qmake-build` 返回非零退出码时
- 构建输出中包含 `Error:`、`fatal error`、`error LNK`、`error C` 等关键字时
- 用户询问"为什么构建失败"、"这个编译错误什么意思"时
- 可执行文件生成失败或启动失败时

## 诊断流程

### 步骤 1：提取关键错误信息
从构建日志的**尾部向上追溯**，提取第一条实质性的错误信息（忽略警告）。重点关注：
- `jom:` / `nmake:` / `make:` 后的错误码
- `fatal error C1083`（找不到文件）
- `error Cxxxx`（编译语法错误）
- `error LNKxxxx`（链接错误）
- `Error: dependent ... does not exist`（Makefile 依赖错误）

### 步骤 2：模式匹配
将提取的错误与下文的**常见错误速查表**进行匹配。

### 步骤 3：执行修复策略
| 问题类型 | 处理方式 |
|---------|---------|
| **可自动修复** | 直接修改配置文件（如 `.pro`、`build.bat`），修复后自动重新调用 `qmake-build` 验证 |
| **需用户确认** | 说明根因和修复方案，询问用户是否执行 |
| **未知问题** | 输出详细分析和排查建议，引导用户提供更多信息 |

---

## 常见错误速查表

### 1. Makefile 依赖路径错误
**错误特征**：
```
Error: dependent '........\Qt\x.x.x\msvc2019_64\include\QtWidgets\QMainWindow' does not exist.
jom: ...\Makefile [release] Error 2
```
路径也可能缺少 `include\` 层级，例如：
```
Error: dependent '........\Qt\x.x.x\msvc2019_64\QtWidgets\QMainWindow' does not exist.
```

**根本原因**：`QMAKE_PROJECT_DEPTH` 配置与 qmake 的目录深度计算冲突，导致生成的 Makefile 中 Qt 系统头文件的依赖路径被解析为错误的相对路径，jom 无法找到对应文件。

**修复方案**：
1. **优先检查** `.pro` 文件中是否包含 `QMAKE_PROJECT_DEPTH = 0`
2. **若有该配置**：将其删除或注释掉，保存后重新构建
3. **若无该配置**：尝试在 `.pro` 文件末尾添加 `QMAKE_PROJECT_DEPTH = 0`，保存后重新构建，观察问题是否解决
4. 若添加或删除后仍失败，检查是否存在其他路径覆盖配置（如手动指定的 `INCLUDEPATH` 与 Qt 系统路径冲突）

**自动修复**：✅ 是（可自动增删 `.pro` 中的 `QMAKE_PROJECT_DEPTH = 0` 并重新触发构建）

---

### 2. build.bat 变量展开顺序错误
**错误特征**：
```
[4/4] Starting application...
    Error: exe not found
```
但实际 `build\release\<项目名>.exe` 已生成。

**根本原因**：`build.bat` 中 `set EXE_PATH=...%PRO_FILE:.pro=%...` 定义在 `PRO_FILE` 赋值之前。批处理在**解析阶段**即展开变量，此时 `PRO_FILE` 为空，导致路径变成 `...\release\.exe`。

**修复方案**：将 `set EXE_PATH` 移至 `PRO_FILE` 赋值并校验之后，例如：
```batch
set "PRO_FILE="
for %%f in (%PROJECT_DIR%*.pro) do if not defined PRO_FILE set "PRO_FILE=%%~nxf"
if not defined PRO_FILE (...) 
echo Found project: %PRO_FILE%

REM 必须放在 PRO_FILE 赋值之后
set EXE_PATH=%BUILD_DIR%\release\%PRO_FILE:.pro=%.exe
```

**自动修复**：✅ 是（修正内嵌脚本中的变量顺序）

---

### 3. MSVC 编译环境缺失
**错误特征**：
```
Error: vcvars64.bat not found
```
或编译阶段出现：
```
'cl' 不是内部或外部命令，也不是可运行的程序
```

**根本原因**：未找到 Visual Studio 的 `vcvars64.bat`，或调用后环境未正确继承。

**修复方案**：运行 `qmake-init` 重新扫描并配置 MSVC 路径。

**自动修复**：❌ 否（需用户从多个候选中选择正确的 MSVC 版本）

---

### 4. Qt / qmake 路径错误
**错误特征**：
```
Error: qmake failed
```
或 qmake 报告找不到 mkspecs、Qt 安装目录等。

**根本原因**：`QT_DIR` 指向了错误路径，或 `bin\qmake.exe` 不存在。

**修复方案**：运行 `qmake-init` 重新扫描并配置 Qt 路径。

**自动修复**：❌ 否（需用户从多个候选中选择正确的 Qt 版本）

---

### 5. 源代码编译错误（一般性）
**错误特征**：
```
jom: ...\Makefile [release] Error 2
```
前面伴随 `fatal error C1083`、`error Cxxxx`、`warning treated as error` 等。

**根本原因**：C++ 源代码存在语法错误、头文件缺失、类型不匹配、API 使用错误等。

**修复方案**：
1. 向上追溯日志，定位**第一个**具体的 `error Cxxxx` 或 `fatal error`
2. 结合错误代码和行号修复源代码

**自动修复**：❌ 否（需要开发者根据具体错误修复代码）

---

### 6. 缺少 Qt 模块依赖
**错误特征**：
```
fatal error C1083: 无法打开包括文件: "QtNetwork/QNetworkAccessManager": No such file or directory
```
或链接阶段：
```
error LNK2019: 无法解析的外部符号 ... Qt5Network.lib / Qt6Network.lib
```

**根本原因**：使用了某个 Qt 模块的类/功能，但未在 `.pro` 文件中声明模块依赖。

**修复方案**：在 `.pro` 文件中添加对应模块：
| 使用的功能 | 添加语句 |
|-----------|---------|
| `QMainWindow`、`QWidget`、`QDialog` 等 | `QT += widgets` |
| `QNetworkAccessManager`、`QTcpSocket` | `QT += network` |
| `QSqlDatabase`、`QSqlQuery` | `QT += sql` |
| `QJsonDocument`、`QJsonObject` | `QT += core`（Qt6 已内置） |
| `QXmlStreamReader` | `QT += xml` |
| `QThreadPool`、`QMutex` | `QT += core` |

**自动修复**：✅ 是（可根据报错头文件推断模块并自动添加到 `.pro`）

---

### 7. MOC（元对象编译器）错误
**错误特征**：
```
moc_mainwindow.cpp(xx): error Cxxxx: ...
```
或 moc 进程本身报错：
```
moc: Cannot open options file specified with @
```

**根本原因**：
- 包含 `Q_OBJECT` 宏的类存在语法错误（如缺少分号、括号不匹配）
- 信号/槽声明格式错误（如参数不匹配、使用了错误的宏）
- 类声明缺少必要的 Qt 头文件包含

**修复方案**：检查报错的头文件（通常是 `mainwindow.h` 等），确保：
- 类声明语法完整
- `Q_OBJECT` 宏在 `public:` 之后
- 信号槽声明格式正确（`signals:` / `private slots:` 等）

**自动修复**：❌ 否（需要开发者修复代码）

---

### 8. UI 文件编译错误
**错误特征**：
```
D:\Qt\x.x.x\msvc2019_64\bin\uic.exe mainwindow.ui
Error: unexpected element ... in mainwindow.ui
```

**根本原因**：`.ui` 文件 XML 格式损坏，或引用了不存在的自定义控件/类。

**修复方案**：使用 Qt Designer 打开 `.ui` 文件检查并修复；若文件严重损坏，可用 Qt Creator 重新生成。

**自动修复**：❌ 否

---

### 9. 链接错误（LNK）
**错误特征**：
```
error LNK2019: 无法解析的外部符号 "..."
error LNK1120: x 个无法解析的外部命令
```

**根本原因**：
- 声明了函数/类但没有实现（如缺少对应的 `.cpp` 文件）
- 调用了外部库但未在 `.pro` 中链接（`LIBS += -L... -l...`）
- 模板类/内联函数定义未放在头文件中

**修复方案**：根据未解析符号名称定位缺失的实现或库依赖。

**自动修复**：❌ 否（需要开发者根据具体符号修复）

---

## 执行策略

调用本技能时，按以下**优先级**依次排查：

1. **检查 build.bat 本身是否有已知 bug**（如 EXE_PATH 变量顺序问题）
2. **检查 .pro 文件是否有已知配置问题**（如 `QMAKE_PROJECT_DEPTH = 0`）
3. **检查环境和工具链路径**（如 vcvars、qmake、jom 缺失或路径错误）
4. **分析具体的编译/链接/MOC/UIC 错误**，匹配速查表

### 修复后验证
- 若执行了自动修复，**必须重新调用 `qmake-build`** 验证是否解决
- 若修复后仍然失败，继续向上追溯日志中的下一个错误
- 若遇到未收录的未知错误，记录错误模式并告知用户需要手动排查

## 备注
- 本技能作为 `qmake-build` 的**前置错误处理层**，构建失败时应**优先调用**，而非直接报错退出
- 所有自动修改操作（如修改 `.pro` 或 `build.bat`）前应向用户说明变更内容
- 对于需要修改用户源代码的编译错误，仅提供分析和修复建议，不擅自修改 `.cpp` / `.h` 文件
