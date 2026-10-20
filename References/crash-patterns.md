# 常见崩溃模式与修复速查表

> 本速查表用于在崩溃分诊后快速匹配根因模式。
> 当 REA/Ghidra 不可用走降级分支时，本表仍可基于异常码/信号、寄存器、调用栈做粗粒度判断。

## 1. 空指针解引用

| 项 | 值 |
|---|---|
| Windows 异常码 | `0xC0000005`（读访问违例） |
| Linux 信号 | `SIGSEGV` |
| 典型特征 | 崩溃指令访问的虚拟地址接近 `0x0`（如 `0x0000000000000008`、`0x00007ff600000000` 附近小偏移） |
| 根因 | 指针未初始化、函数返回空指针未校验、对象提前释放、vtable 指针损坏 |
| 修复 | 所有指针解引用前加空值校验；对返回指针的函数调用后立即判空；失败路径走错误处理，禁止继续执行 |

```cpp
// 修复前
void process(Data* ptr) {
    ptr->value = 0;
}

// 修复后
void process(Data* ptr) {
    if (ptr == nullptr) {
        return; // 或返回错误码
    }
    ptr->value = 0;
}
```

## 2. 内存释放后使用（UAF）

| 项 | 值 |
|---|---|
| Windows 异常码 | `0xC0000005`（通常是写访问违例） |
| Linux 信号 | `SIGSEGV` |
| 典型特征 | 访问地址落在已释放堆块上；堆块头常被 freelist 覆写为 `0x...21`、`0x...29` 这类小整数；调用栈中往往能看到 `free` 之后又 `delete obj` |
| 根因 | 释放后指针未置空；对象生命周期管理混乱；跨模块释放 |
| 修复 | 释放后立即 `nullptr`；用 `std::unique_ptr` / `std::shared_ptr` 托管；统一分配释放边界 |

```cpp
// 修复前
delete obj;
obj->doSomething();

// 修复后
delete obj;
obj = nullptr;
```

## 3. 数组 / 缓冲区越界

| 项 | 值 |
|---|---|
| Windows 异常码 | `0xC0000005` |
| Linux 信号 | `SIGSEGV` |
| 典型特征 | 访问地址紧跟在数组/缓冲区末尾之后；写越界会破坏相邻堆块元数据，可能延后在 `malloc/free` 才崩溃 |
| 根因 | 索引未做边界校验；字符串操作溢出；`memcpy` 长度参数错 |
| 修复 | 索引前校验范围；用 `std::array` / `std::vector`；字符串操作用带长度参数的安全函数 |

```cpp
// 修复前
int arr[10];
arr[index] = 0;

// 修复后
int arr[10];
if (index >= 0 && index < 10) {
    arr[index] = 0;
}
```

## 4. 栈溢出

| 项 | 值 |
|---|---|
| Windows 异常码 | `0xC00000FD`（STACK_OVERFLOW） |
| Linux 信号 | `SIGSEGV`，且 PC/SP 落在栈保护区（`[stack]` 段末端） |
| 典型特征 | 调用栈极深且重复（递归）；函数内有大体积局部数组（`char buf[8192]`） |
| 根因 | 无限递归；超大局部变量；栈破坏 |
| 修复 | 递归加深度限制或改迭代；大变量改堆分配；必要时调大线程栈（仅临时） |

```cpp
// 修复前
void recursive(int n) {
    char buf[8192];
    recursive(n + 1);
}

// 修复后
void recursive(int n, int depth = 0) {
    if (depth > MAX_DEPTH) return;
    auto buf = std::make_unique<char[]>(8192);
    recursive(n + 1, depth + 1);
}
```

## 5. 整数除零

| 项 | 值 |
|---|---|
| Windows 异常码 | `0xC0000094`（INT_DIVIDE_BY_ZERO） |
| Linux 信号 | `SIGFPE` |
| 典型特征 | 崩溃指令为 `idiv` / `div`，除数寄存器值为 0 |
| 根因 | 除数未校验零值；边界条件下除数为 0 |
| 修复 | 除法前对除数做零值校验 |

```cpp
// 修复前
int result = a / b;

// 修复后
if (b == 0) {
    return -1;
}
int result = a / b;
```

## 6. 堆损坏

| 项 | 值 |
|---|---|
| Windows 异常码 | `0xC0000374`（HEAP_CORRUPTION） |
| Linux 信号 | `SIGABRT`，且调用栈在 `malloc` / `free` / `abort` / `__libc_message` 内 |
| 典型特征 | 崩溃点不在"出错那行"，而是延迟到下一次堆操作；`free()` 报 corrupted top size / double free / invalid pointer |
| 根因 | 堆越界写入；重复释放；野指针写入 |
| 修复 | 排查所有堆写入长度；禁止重复释放；用 Valgrind / AddressSanitizer / PageHeap 定位真正越界点 |

---

## 模式匹配流程

1. 先看异常码 / 信号 → 缩小到上表 6 类之一；
2. 看崩溃指令的操作数地址与寄存器值 → 验证假设；
3. 看调用栈上下文 → 确认是"当场崩"还是"延迟崩"（堆损坏常延迟）；
4. 若 REA/Ghidra 可用，再用反汇编与伪代码确认具体逻辑错误。
