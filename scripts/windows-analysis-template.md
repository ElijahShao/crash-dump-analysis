# Windows 崩溃分析报告

> 本模板由 `cross-platform-crash-reverse-analysis` Skill 输出。
> 若 REA/Ghidra 不可用，"反汇编上下文 / 伪代码 / 源码定位"三节写 **"REA 不可用，未生成"**，并在开头显著位置标注降级状态。

## 0. 工具链状态

| 组件 | 状态 | 备注 |
|---|---|---|
| mcp-windbg | ✅ 正常 / ❌ 失败 | session_id：`<cdb-xxxxxxxx>` |
| REA MCP | ✅ 正常 / ❌ 失败 | |
| Ghidra provider | ✅ available / ❌ unavailable | |
| 是否走降级分支 | 是 / 否 | 若"是"，说明失败原因 |

---

## 1. 基本信息

| 项 | 值 |
|---|---|
| 平台 | Windows |
| 可执行文件 | `target.exe`（版本：xxx） |
| 转储文件 | `crash.dmp` |
| 符号文件（PDB） | 有 / 无 |
| 异常代码 | `0xC0000005` |
| 异常描述 | 访问违例（读 / 写） |
| 崩溃地址（绝对） | `0x00007FF612345678` |
| 崩溃所属模块 | `target.exe` |
| 模块加载基址 | `0x00007FF612330000` |

> RVA 由 REA/Ghidra 归一化后填入，不在主流程手算。

## 2. 崩溃调用栈（`mcp-windbg.run_cdb_command` 输出 `kP`）

```
#0 0x00007FF612345678 target.exe!function_name
#1 0x00007FF612345000 target.exe!caller_function
#2 ...
```

## 3. 崩溃点详细分析

### 3.1 汇编级上下文（REA/Ghidra 提供）

```asm
; 前 5 行
0x00012330: mov rax, [rcx]
0x00012333: add rax, 0x8
; ---- 崩溃点 ----
0x00012345: mov rdx, [rax]   ; 崩溃指令
; ----
; 后 5 行
0x00012348: call 0x00012400
```

> 若 REA 不可用，本节写：**"REA 不可用，未生成反汇编上下文。以下为 mcp-windbg `u <ip> L10` 的原始输出（如有）。"**

### 3.2 反编译伪代码（REA + Ghidra `batch_decompile`）

```c
void function_name(Param1 *param1) {
    // ...
    // 崩溃位置
    value = param1->field->data;
    // ...
}
```

### 3.3 源码定位（mcp-windbg `ln <ip>` / `.lines -e`）

- **文件**：`src/module/file.cpp`
- **行号**：128
- **代码片段**：

```cpp
// 第 128 行
int value = obj->field->data;
```

> 无 PDB 时写：**"无 PDB，函数名为 Ghidra 自动识别，源码行号不可用。"**

## 4. 根因分析

[结合异常码、寄存器值、崩溃指令操作数、调用栈，说明崩溃触发的完整逻辑链路。]

[匹配 `references/crash-patterns.md` 中的 6 类模式，给出模式名。]

## 5. 修复方案

### 5.1 问题描述

[精准描述代码中的逻辑错误。]

### 5.2 修复代码

```cpp
// 修复后的代码片段
```

### 5.3 风险与边界

- 修复影响的范围：
- 可能的边界情况：
- 性能影响评估：

### 5.4 验证方法

1. 构造触发崩溃的输入用例；
2. 验证修复后不再崩溃；
3. 验证正常功能不受影响。

---

## 附录

### A. 寄存器状态（`r` 命令）

| 寄存器 | 值 |
|---|---|
| rax | `0x0000000000000000` |
| rcx | `0x000000FEDCBA9876` |
| rdx | ... |
| rip | `0x00007FF612345678` |

### B. 模块加载基址（`lm` 命令）

| 模块 | 基址 |
|---|---|
| target.exe | `0x00007FF612330000` |
| ... | ... |

### C. 分析工具版本

- mcp-windbg：xxx
- REA：xxx
- Ghidra：xxx
- cdb.exe：xxx
