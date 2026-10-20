# 分析前检查清单

> 本清单在 SKILL.md 闸门 0 / 闸门 1 / 闸门 2 处逐项核对。
> 闸门 1（调试器 MCP 连通性）失败 = 硬失败，终止；
> 闸门 2（REA/Ghidra）失败 = 软失败，走降级分支。

## 1. 文件检查

- [ ] 崩溃转储文件存在且可读（Windows `.dmp/.mdmp/.hdmp`；Linux `core.<pid>`）
- [ ] 对应可执行文件存在且可读（Windows `.exe`；Linux ELF）
- [ ] 转储与可执行文件为**同一编译版本**（编译时间戳、构建号一致）
- [ ] 符号文件已准备（Windows PDB；Linux DebugInfo）—— 优先
- [ ] 源码已准备（可选，用于源码映射）
- [ ] 所有文件路径无中文、无空格、无特殊字符

## 2. 闸门 1：调试器 MCP 连通性（硬失败即终止）

### Windows 平台
- [ ] Claude Code MCP 配置中已注册 `mcp-windbg`
- [ ] 本机已安装 Debugging Tools for Windows，`cdb.exe` 可执行
- [ ] 调用 `mcp-windbg.list_dumps` 返回正常（非 MCP 错误、非 cdb 启动失败）
- [ ] `_NT_SYMBOL_PATH` 已配置（指向微软符号服务器 + 本地缓存目录）

> 任意一项不通过 → 终止 Skill，向用户输出 mcp-windbg 连接异常提示。

### Linux 平台
- [ ] Claude Code MCP 配置中已注册 `gdb-mcp`
- [ ] Python 版本 ≥ 3.12（gdb-mcp 硬性要求）
- [ ] `which gdb` 可执行，GDB 版本 ≥ 12
- [ ] 调用 `gdb-mcp.list_sessions` 返回正常 JSON 数组（即使为空也算通过）

> 任意一项不通过 → 终止 Skill，向用户输出 gdb-mcp 连接异常提示。

## 3. 闸门 2：REA + Ghidra（软失败可降级）

- [ ] Node.js 版本符合 REA 要求（22.19+ / 24.11+ / 26+）
- [ ] npm 正常可用
- [ ] `rea` 已全局安装：`rea --version`
- [ ] Ghidra 已安装，JDK 版本匹配（推荐 JDK 21+）
- [ ] 环境变量 `GHIDRA_INSTALL_DIR` / `JAVA_HOME` 已设置
- [ ] Linux/macOS 上 PATH 含 `ps`（procps-ng）
- [ ] 调用 `rea.providers list` 中 `ghidra` 状态为 `available`
- [ ] 试调用 `rea.open_binary(可执行文件, provider="ghidra")` 不超时

> 任意一项不通过 → **不终止**，走降级分支：
> - 报告头部写"REA/Ghidra 不可用"警告；
> - 不打开二进制、不反汇编、不算 RVA、不定位源码行；
> - 仅基于 Dump/Core 分诊结果输出报告。

## 4. 权限检查

- [ ] 拥有目标软件的合法分析授权
- [ ] 对文件目录有读写权限
- [ ] 调试工具具备足够权限（Windows 管理员 / Linux root 或对应用户）
- [ ] Linux 上 core 文件未被 `ulimit -c 0` 禁用；core_pattern 指向可写目录

## 5. 预校验

- [ ] 转储文件可被调试器正常打开（`open_cdb_dump` / `gdb-mcp.call core-file`）
- [ ] 可执行文件架构与调试器一致（x86_64 ELF 不能用 arm64 GDB 分析）
- [ ] 模块基址与地址空间无异常（Windows `lm` 输出正常；Linux `info sharedlibrary` 输出正常）
