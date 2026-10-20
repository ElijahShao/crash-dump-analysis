# Windows 工具链配置与调用手册
> 严格遵循 mcp-windbg 官方 API 规范：[https://github.com/svnscha/mcp-windbg](https://github.com/svnscha/mcp-windbg)
> 本工具链仅负责 Dump 分诊与原始数据提取，**RVA 计算、二进制逆向、源码映射统一交由 REA MCP 内部完成**

## 工具链组成
- 调试器：mcp-windbg（基于 WinDbg/CDB 引擎，用户态 Dump 分析）
- 逆向编排：REA (rea-agents)
- 反汇编引擎：Ghidra

## 环境配置要求
### 1. mcp-windbg MCP 服务注册
在 Claude Code MCP 配置文件中添加：
```json
{
  "mcp-windbg": {
    "command": "npx",
    "args": ["-y", "mcp-windbg@latest"]
  }
}
```
**验证方式**：调用 `mcp-windbg.list_dumps` 可正常返回目录内 Dump 文件列表。

### 2. REA + Ghidra 关联配置
1. 安装 Ghidra 并配置 JDK 25 64位
2. 设置系统环境变量：
   ```cmd
   setx REA_GHIDRA_INSTALL_DIR "C:\\tools\\ghidra"
   setx REA_GHIDRA_STARTUP_TIMEOUT_MS 120000
   ```
3. **验证方式**：执行 `rea providers list`，输出中 `ghidra` 状态为 `available`。

## 分步工具调用详解
### 步骤1：连通性预检测（闸门1 · 硬终止）
**对应规则**：调试器 MCP 连接异常则直接终止流程，不打开 Dump 文件
- **工具**：`mcp-windbg.list_dumps`
- **参数**：
  ```json
  {
    "directory": "C:\\path\\to\\dumps"
  }
  ```
- **判定规则**：
  - 调用成功且返回有效文件列表 → 连通性正常，进入下一步
  - 调用失败/超时/报错 → 输出提示「mcp-windbg 工具连接异常，请检查MCP服务配置后重试」，**终止整个 Skill 流程**

---

### 步骤2：Dump 文件解析（mcp-windbg）
**核心职责**：仅提取崩溃原始数据，不手动计算 RVA
- **工具**：`mcp-windbg.open_cdb_dump`
- **参数**：
  ```json
  {
    "dump_path": "C:\\path\\to\\crash.dmp",
    "exe_path": "C:\\path\\to\\target.exe",
    "symbol_path": "C:\\path\\to\\target.pdb",
    "run_analyze": true,
    "include_stack": true,
    "include_registers": true,
    "include_modules": true
  }
  ```
- **提取关键字段（仅原始数据）**：
  - `exception_code`：异常代码
  - `faulting_address`：崩溃绝对地址
  - `faulting_module`：崩溃所属模块
  - `module_base`：模块加载基址
  - `stack_trace`：完整调用栈
  - `registers`：寄存器状态
  - `session_id`：会话唯一标识，用于后续命令与会话关闭

> ⚠️ **强制约束**：本步骤禁止手动计算 `faulting_address - module_base`，RVA 统一由 REA 内部完成地址归一化。

---

### 步骤3：扩展调试命令（可选）
**工具**：`mcp-windbg.run_cdb_command`
- **参数**：
  ```json
  {
    "session_id": "cdb-xxxxxx",
    "command": "lmvm target"
  }
  ```
- **适用场景**：
  - 查看模块详细版本与时间戳信息
  - 补充执行 `!heap -s`、`!analyze -v` 等扩展诊断命令
  - 验证符号文件加载状态

---

### 步骤4：REA 逆向分析（地址归一化由 REA 内部完成）
#### 4.1 二进制加载与连通性检测（闸门2 · 降级分支）
**对应规则**：REA/Ghidra 异常不终止流程，降级为纯 Dump 分析
- **工具**：`rea.open_binary`
- **参数**：
  ```json
  {
    "provider": "ghidra",
    "binary_path": "C:\\path\\to\\target.exe",
    "pdb_path": "C:\\path\\to\\target.pdb"
  }
  ```
- **判定规则**：
  - 加载成功 → 进入完整逆向定位流程
  - 加载失败 → 输出提示「REA 服务或 Ghidra 引擎连接异常，本次仅执行崩溃转储基础分析，不进行二进制逆向与源码定位」，跳过后续逆向步骤，生成降级报告

#### 4.2 崩溃点定位（REA 内部自动计算 RVA）
- **工具**：`rea.inspect_native_instruction`
- **参数**：
  ```json
  {
    "provider": "ghidra",
    "binary_path": "C:\\path\\to\\target.exe",
    "address": "0x00007FF612345678",
    "context_lines": 10
  }
  ```
- **输出产物**：
  - 所属函数名称与起始地址
  - 崩溃点汇编指令及前后 10 行上下文
  - 对应 Ghidra 反编译伪代码片段
  - REA 内部计算完成的 RVA 值
  - 存在 PDB 时返回源码文件路径与行号

#### 4.3 函数深度逆向分析
- **工具**：`rea.analyze_function`
- **参数**：
  ```json
  {
    "provider": "ghidra",
    "binary_path": "C:\\path\\to\\target.exe",
    "function_address": "0x00012300"
  }
  ```
- **输出产物**：
  - 完整函数反编译伪代码
  - 函数内交叉引用关系
  - 字符串与常量引用列表

#### 4.4 源码级映射（可选）
- **工具**：`rea.address_to_source`
- **参数**：
  ```json
  {
    "provider": "ghidra",
    "binary_path": "C:\\path\\to\\target.exe",
    "address": "0x00007FF612345678"
  }
  ```

---

### 步骤5：会话关闭
分析完成后必须依次关闭会话，释放系统资源：
```json
// 关闭 WinDbg CDB 会话
{
  "tool": "mcp-windbg.close_cdb_session",
  "params": {
    "session_id": "cdb-xxxxxx"
  }
}

// 关闭 REA 二进制分析会话
{
  "tool": "rea.close_binary",
  "params": {
    "binary_path": "C:\\path\\to\\target.exe"
  }
}
```