# Linux 工具链配置与调用手册

> 上游官方文档：
> - gdb-mcp：https://github.com/hnmr293/gdb-mcp
> - REA：https://github.com/morluto/rea
> - Ghidra：https://github.com/NationalSecurityAgency/ghidra

## 工具链组成

| 角色 | 工具 | 本 Skill 中承担的职责 |
|---|---|---|
| 调试器 | gdb-mcp（基于 GDB MI 接口） | **仅做 Core 分诊**：启动 GDB、加载 ELF 与 Core、跑 `bt` / `info registers` / `info sharedlibrary`。**不计算 PC 相对偏移**。 |
| 逆向编排 | REA（`rea-agents` npm 包） | 打开 ELF，完成虚拟地址归一化、函数反查、反汇编、伪代码。 |
| 反汇编引擎 | Ghidra | REA 的 provider，对 ELF 做静态分析。 |

## 环境配置

### 1. 前置依赖

- Linux x64（Ubuntu 20.04+ / CentOS 8+ 均可）；
- **Python 3.12+**（gdb-mcp 硬性要求）；
- GDB 12+（`which gdb` 可执行）；
- Node.js 22.19+ / 24.11+ / 26+（REA 要求）；
- Ghidra + JDK 21+；
- REA 在 Linux 上还要求 PATH 里有 `ps`（procps-ng），用于进程所有权检查。

### 2. gdb-mcp 在 Claude Code 中注册

仓库源码安装：
```bash
git clone https://github.com/hnmr293/gdb-mcp.git
cd gdb-mcp
pip install -r requirements.txt
pip install -e .
```

或直接使用发布的命令：
```bash
pip install gdb-mcp
```

MCP 配置（`~/.claude/mcp.json`）：
```json
{
  "gdb-mcp": {
    "command": "gdb-mcp"
  }
}
```

或：
```json
{
  "gdb-mcp": {
    "command": "python",
    "args": ["-m", "gdb_mcp"]
  }
}
```

### 3. REA + Ghidra 配置

```bash
npx rea-agents setup
```

环境变量：
```bash
export REA_ANALYSIS_PROVIDER=ghidra
export GHIDRA_INSTALL_DIR=/opt/ghidra_12.1.4_PUBLIC
export JAVA_HOME=/usr/lib/jvm/java-21-openjdk
export REA_GHIDRA_STARTUP_TIMEOUT_MS=120000
```

写入 `~/.bashrc` 持久化。

验证：
```bash
rea providers list
```

---

## 分步工具调用（严格按官方 API）

gdb-mcp 官方只有 4 个工具：`open` / `call` / `close` / `list_sessions`。本 Skill 严格使用这 4 个，不臆造 `gdb-mcp.open_core` 之类的接口。

### 步骤 1：连通性闸门（必做）

**工具**：`gdb-mcp.list_sessions`

```json
{}
```

- 返回 JSON 数组（即使为空 `[]`）→ 闸门通过。
- 抛错（MCP 未注册 / gdb 不可执行 / Python 版本低）→ 按 SKILL.md 分支 B.1 的提示终止流程。

### 步骤 2：启动 GDB 会话

**工具**：`gdb-mcp.open`

```json
{
  "timeout": 300
}
```

返回形如：
```json
{ "id": "550e8400-e29b-41d4-a716-446655440000" }
```

后续所有命令都要带上这个 `id`。

### 步骤 3：加载 ELF 与 Core（仅崩溃分析）

**工具**：`gdb-mcp.call`

依次执行：
```json
{ "id": "550e8400-...", "command": "file /opt/app/bin/target" }
```
```json
{ "id": "550e8400-...", "command": "core-file /var/coredumps/core.1234" }
```
```json
{ "id": "550e8400-...", "command": "info program" }
```
```json
{ "id": "550e8400-...", "command": "bt" }
```
```json
{ "id": "550e8400-...", "command": "info registers" }
```
```json
{ "id": "550e8400-...", "command": "info sharedlibrary" }
```

> **本步不计算 PC 偏移**。只记录：
> - 信号（`SIGSEGV` / `SIGABRT` / `SIGFPE`）
> - 崩溃 PC 绝对地址 `pc`
> - 崩溃所属模块 `crash_module`（可执行文件或某个 `.so`）
> - 该模块加载基址（从 `info sharedlibrary` 输出解析）
> - 完整 backtrace 与寄存器原文

### 步骤 4：REA 连通性 + 打开 ELF

**工具**：`rea.open_binary`

```json
{
  "binary_path": "/opt/app/bin/target",
  "provider": "ghidra"
}
```

- 成功 → 步骤 5。
- 失败 → 走降级分支（见 SKILL.md 分支 B.3）。

### 步骤 5：偏移归一化 + 函数反查（交由 REA）

**工具**：`rea.resolve_containing_procedure` 或 `rea.inspect_native_instruction`

```json
{
  "binary_path": "/opt/app/bin/target",
  "provider": "ghidra",
  "address": "0x0000555555551234",
  "module_base": "0x0000555555550000"
}
```

REA/Ghidra 内部完成：
1. ELF 虚拟地址 → 相对地址归一化；
2. 反查包含该地址的函数；
3. 返回函数名、起始地址、崩溃指令反汇编上下文。

> **不要**自己在脑子里或脚本里算 `pc - module_base`。这一步是 REA 的职责。

### 步骤 6：函数档案与伪代码

**工具**：`rea.analyze_function` / `rea.batch_decompile`

```json
{
  "binary_path": "/opt/app/bin/target",
  "provider": "ghidra",
  "procedure": "0x00001230"
}
```

### 步骤 7：源码行号（若有 DebugInfo）

**工具**：`gdb-mcp.call`

```json
{ "id": "550e8400-...", "command": "info line *0x0000555555551234" }
```

或带 DebugInfo 时直接：
```json
{ "id": "550e8400-...", "command": "bt -full" }
```

源码行号由 GDB 侧完成，不依赖 REA。

### 步骤 8：关闭会话

**工具**：`gdb-mcp.close`

```json
{ "id": "550e8400-e29b-41d4-a716-446655440000" }
```

---

## 常用 GDB 命令速查（通过 `gdb-mcp.call` 执行）

| 目的 | 命令 |
|---|---|
| 完整调用栈 | `bt` / `bt full` / `thread apply all bt` |
| 寄存器 | `info registers` / `i r` |
| 信号与程序状态 | `info program` |
| 共享库加载基址 | `info sharedlibrary` / `info proc mappings` |
| 崩溃点反汇编 | `x/10i $pc-20` |
| 源码行反查 | `info line *0x<pc>` |
| 查看崩溃时栈内存 | `x/32gx $rsp` |
| 线程列表 | `info threads` |

---

## 故障排查

| 现象 | 处理 |
|---|---|
| `list_sessions` 抛错 | 检查 Python 版本 ≥3.12、`which gdb`、MCP 是否重启。 |
| `open` 后 `file` 报 `not in executable format` | ELF 架构与 GDB 不匹配（如 arm64 二进制跑在 x86 GDB 上），换对应架构 GDB。 |
| `core-file` 报 `is not a core dump` | core 文件被截断，或与 ELF 不是同版本；重新生成 core。 |
| `bt` 全是 `??` | 缺 DebugInfo；走降级分支，或在报告中标注"符号不可用"。 |
| `rea.open_binary` 报 `provider unavailable` | Ghidra 未安装、`GHIDRA_INSTALL_DIR` 错、或 PATH 缺 `ps`；走降级分支。 |
| `open_binary` 超时 | 调大 `REA_GHIDRA_STARTUP_TIMEOUT_MS`。 |
| 反查函数错位 | 确认 ELF 与 core 是同一编译版本；strip 过的二进制无法映射源码。 |
