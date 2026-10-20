---
name: cross-platform-crash-reverse-analysis
description: 跨平台崩溃逆向分析Skill。通过标准化斜杠命令触发，Windows联合mcp-windbg+REA+Ghidra解析Dump与EXE，Linux联合gdb-mcp+REA+Ghidra解析Core与ELF；支持调试器MCP硬终止闸门、REA异常降级分析，RVA计算统一由REA处理，所有MCP调用严格遵循官方API规范。
version: 2.1.0
author: Reverse Engineering Team
tags: [crash-analysis, reverse-engineering, ghidra, rea, windbg, gdb, mcp]
mcp-dependencies: [mcp-windbg, gdb-mcp, rea]
tool-dependencies: [Ghidra, WinDbg/CDB, GDB]
upstream-repos:
  mcp-windbg: [https://github.com/svnscha/mcp-windbg](https://github.com/svnscha/mcp-windbg)
  gdb-mcp: [https://github.com/hnmr293/gdb-mcp](https://github.com/hnmr293/gdb-mcp)
  rea: [https://github.com/morluto/rea](https://github.com/morluto/rea)
  ghidra: [https://github.com/NationalSecurityAgency/ghidra](https://github.com/NationalSecurityAgency/ghidra)
trigger-command: /crash-analyze
---

## 概述

本Skill驱动多工具链完成**命令解析 → 转储分析 → 二进制逆向工程 → 源码级定位 → 修复方案输出**的全流程标准化分析。通过固定格式的斜杠命令触发，支持「全量绝对路径」与「项目根目录相对路径」两种调用形式，根据转储文件自动识别目标平台：

- **Windows平台**：`mcp-windbg` 解析Dump → `REA`调度`Ghidra`逆向EXE → 地址归一化与源码匹配 → 输出修复方案
- **Linux平台**：`gdb-mcp` 解析Core → `REA`调度`Ghidra`逆向ELF → 地址归一化与源码匹配 → 输出修复方案

RVA/相对偏移计算统一由 REA 内部完成，调试器 MCP 仅负责崩溃转储分诊与原始数据提取。

## 触发条件与命令格式

### 统一触发命令

本Skill仅通过标准化斜杠命令触发，命令名为 **`/crash-analyze`**，支持两种合法调用形式。

---

### 形式一：全量绝对路径模式（任意目录下执行）

#### 语法

```
/crash-analyze <Dump文件绝对路径> <可执行文件绝对路径> [源码根目录绝对路径]
```

#### 参数说明

| 参数 | 必填 | 说明 |
| --- | --- | --- |
| Dump文件绝对路径 | 是 | Windows 支持 `.dmp`/`.mdmp`/`.hdmp`；Linux 支持 `core`/`core.*` 格式 |
| 可执行文件绝对路径 | 是 | Windows 为 `.exe` 二进制；Linux 为 ELF 可执行文件 |
| 源码根目录绝对路径 | 否 | 显式指定源码项目根目录，用于源码行映射；不传则跳过源码映射步骤 |

#### 示例

```
# Windows 全量模式
/crash-analyze C:\dumps\app-crash.dmp C:\build\target.exe C:\src\project-root

# Linux 全量模式
/crash-analyze /var/crash/core.1234 /opt/app/target /home/dev/project-root
```

---

### 形式二：项目根目录相对路径模式（当前目录为源码根目录时执行）

#### 语法

```
/crash-analyze <Dump文件路径> <可执行文件路径>
```

#### 规则说明

1. 执行命令时的**当前工作目录默认为源码项目根目录**，自动用于源码映射，无需额外传入
2. Dump文件、可执行文件支持**相对路径（相对于当前工作目录）** 或 **绝对路径**
3. 若传入绝对路径，则直接使用绝对路径，不受当前目录影响

#### 示例

```
# 当前目录为 C:\src\project-root，Dump与EXE在子目录下
/crash-analyze dumps\crash.dmp build\Release\target.exe

# 当前目录为 /home/dev/project-root，混合相对+绝对路径
/crash-analyze ./build/core.5678 /opt/debug/target
```

---

### 平台自动识别规则

无需手动指定平台参数，Skill 按以下优先级自动判定分析分支：

1. 后缀匹配：`.dmp`/`.mdmp`/`.hdmp` → Windows 分支；`core`/`core.数字` → Linux 分支
2. 可执行文件格式辅助校验：Windows `.exe` / Linux 无后缀ELF
3. 识别失败时抛出参数错误提示，终止流程

### 参数校验规则

触发命令后先执行强制校验，不通过则直接返回错误提示、不进入分析流程：

1. 参数数量不足2个 → 提示「参数错误，请按格式传入Dump文件路径与可执行文件路径」
2. Dump文件不存在/不可读 → 提示「Dump文件不存在或无读取权限」
3. 可执行文件不存在/不可读 → 提示「可执行文件不存在或无读取权限」
4. 显式传入源码目录但目录不存在 → 提示「源码根目录不存在」

## 前置环境检查

执行完整分析前必须通过检查，详细清单见 `scripts/common-checklist.md`：

1. **MCP服务注册**：Claude Code 已配置对应平台调试器MCP与 `rea` MCP服务器且运行正常
2. **Ghidra关联**：REA 已正确关联 Ghidra 安装目录，`rea providers list` 中 ghidra 状态为 `available`
3. **文件一致性**：Dump/Core 与可执行文件为**同一编译版本**，优先提供符号文件（PDB/Debug Info）
4. **路径规范**：所有文件路径无中文、无空格、无特殊字符，避免编码与解析异常

## 核心工作流总览

```
命令解析与路径归一化
        │
        ▼
闸门0：文件存在性与版本一致性校验
        │
        ▼
闸门1：调试器MCP连通性检测（硬失败→终止流程）
        │
        ▼
阶段一：调试器转储分诊（仅提取原始地址，不计算RVA）
        │
        ▼
闸门2：REA+Ghidra连通性检测（软失败→降级分支）
        │
        ├─ 正常 → REA地址归一化 → 二进制逆向 → 崩溃点定位 → 源码映射 → 完整分析报告
        │
        └─ 异常 → 跳过逆向与源码定位 → 仅输出转储分诊报告（标注REA不可用）
        │
        ▼
标准化报告输出
```

### 步骤0：命令解析与路径归一化

1. 解析 `/crash-analyze` 命令参数，区分全量模式与项目根目录模式
2. 相对路径统一转换为绝对路径：项目根目录模式下，以当前工作目录为基准拼接
3. 自动识别目标平台（Windows/Linux）
4. 校验所有路径合法性，不通过则输出对应错误提示并终止

### 闸门1：调试器MCP连通性检测（硬终止闸门）

- **Windows分支**：调用 `mcp-windbg.list_dumps` 传入Dump所在目录，验证服务连通性
  - 连通失败 → 输出提示：「mcp-windbg 工具连接异常，请检查MCP服务配置后重试」，终止Skill流程，不打开Dump文件
  - 连通正常 → 进入Dump分诊阶段
- **Linux分支**：调用 `gdb-mcp.list_sessions` 验证服务连通性
  - 连通失败 → 输出提示：「gdb-mcp 工具连接异常，请检查MCP服务配置后重试」，终止Skill流程，不加载Core文件
  - 连通正常 → 进入Core分诊阶段

### 阶段一：调试器转储分诊（仅提取原始数据）

#### 分支A：Windows平台

详细步骤与官方工具调用参数见 `references/windows-toolchain.md`

1. **Dump分诊**：调用 `mcp-windbg.open_cdb_dump` 加载Dump，执行自动分析
2. **原始数据留存**：仅提取异常代码、崩溃绝对地址、调用栈、模块基址、寄存器，**不手动计算RVA**，RVA计算统一交由REA内部处理

#### 分支B：Linux平台

详细步骤与官方工具调用参数见 `references/linux-toolchain.md`

1. **Core分诊**：调用 `gdb-mcp.open` 创建会话，通过 `call` 执行 `file` + `core-file` 加载目标文件
2. **原始数据留存**：仅提取信号类型、PC绝对地址、调用栈、库基址、寄存器，**不手动计算偏移**，地址归一化统一交由REA内部处理

### 闸门2：REA+Ghidra连通性检测（降级闸门）

1. 调用 `rea.open_binary` 尝试加载可执行文件，检测REA服务与Ghidra引擎可用性
2. **连接正常**：进入完整逆向分析流程
3. **连接异常**：
   - 输出提示：「REA 服务或 Ghidra 引擎连接异常，本次仅执行崩溃转储基础分析，不进行二进制逆向与源码定位」
   - 跳过所有逆向、反编译、源码映射步骤
   - 按报告模板生成**降级版崩溃分析报告**，标注「REA不可用」，仅包含转储基础信息、调用栈、寄存器、模块列表

### 阶段二：REA 逆向分析与崩溃定位（正常分支）

#### 分支A：Windows平台

1. **地址归一化与函数定位**：调用 `rea.inspect_native_instruction`，传入崩溃绝对地址与模块信息，由REA内部完成RVA计算、所属函数定位、汇编上下文提取
2. **二进制逆向**：调用 `rea.analyze_function` 生成崩溃函数的C风格伪代码、交叉引用
3. **源码映射**：存在符号文件与源码目录时，调用 `rea.address_to_source` 映射到具体源码文件与行号
4. **根因与修复**：匹配崩溃模式库，输出修复方案与验证建议

#### 分支B：Linux平台

1. **地址归一化与函数定位**：调用 `rea.inspect_native_instruction`，传入PC绝对地址，由REA内部完成相对偏移计算、函数定位、汇编上下文提取
2. **二进制逆向**：调用 `rea.batch_decompile` 对崩溃所在函数范围反编译
3. **源码映射**：存在调试符号与源码目录时，通过REA或gdb-mcp执行 `info line *地址` 映射源码行号
4. **根因与修复**：匹配崩溃模式库，输出修复方案与验证建议

### 会话关闭

分析完成后必须关闭对应调试器会话与REA二进制会话，释放资源：

- Windows：调用 `mcp-windbg.close_cdb_session`
- Linux：调用 `gdb-mcp.close`
- REA：关闭二进制分析会话

## 输出报告规范

### 输出报告模板

最终**必须输出**结构化分析报告，**严格遵循**对应平台模板：

- Windows报告模板：`scripts/windows-analysis-template.md`
- Linux报告模板：`scripts/linux-analysis-template.md`

### 报告分级规则

1. **完整报告（REA正常）**：包含全部证据链
2. **降级报告（REA异常）**：仅包含崩溃基本信息、调用栈、寄存器、模块列表；反汇编、伪代码、源码定位章节标注「REA服务异常，未执行逆向分析」

完整报告必须包含完整证据链：

1. 崩溃基本信息与调用栈
2. 崩溃点汇编上下文（前后各5行）
3. Ghidra反编译伪代码片段
4. 源码定位结果（如有）
5. 根因详细分析
6. 可落地的修复代码与验证方法

## 常见崩溃模式速查

完整崩溃模式、根因、修复要点见 `references/crash-patterns.md`，覆盖：

- 空指针解引用（0xC0000005 / SIGSEGV）
- 内存释放后使用（UAF）
- 数组/缓冲区越界
- 栈溢出
- 整数除零
- 堆损坏

## 注意事项

1. **会话管理**：调试器与REA会话使用完毕后必须关闭，释放资源
2. **版本匹配**：严格保证转储与可执行文件版本一致，否则地址映射完全失效
3. **符号优先**：始终优先使用符号文件，无符号时明确标注函数为自动识别
4. **超时处理**：大文件逆向分析可分步执行，先分析崩溃所在模块
5. **合规声明**：仅对拥有合法授权的软件进行逆向分析，遵守相关法律法规
6. **调用规范**：所有MCP工具调用严格遵循对应官方仓库的API定义，禁止使用非官方接口
