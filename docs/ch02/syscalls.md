# 系统调用路径

用户态进入内核的同步路径是系统调用：不经普通 `call` 的返回地址约定，而经架构定义的陷入指令，带着约定好的号与参数切换特权级。本页给出 Linux x86-64 用户态的寄存器约定与和普通函数调用的对照；异步打断见 [陷阱、信号与异步帧](exceptions-signals.md)。

## syscall 指令与寄存器约定

Linux x86-64 用户态用 `syscall` 指令陷入。调用号放在 `%rax`；参数依次为 `%rdi`、`%rsi`、`%rdx`、`%r10`、`%r8`、`%r9`（注意第四参数是 `%r10`，不是 SysV 函数调用的 `%rcx`——因为 `syscall` 本身会把返回地址写入 `%rcx`，并把 `rflags` 写入 `%r11`）。

返回值在 `%rax`：非负为成功结果，`-4095` 到 `-1` 的范围在 libc 包装里常被翻译成 `errno`。手写裸 `syscall` 时通常直接判断 `%rax` 的符号。

最小退出示例：

```asm
    movq    $60, %rax       # SYS_exit
    movq    $0, %rdi        # status
    syscall
```

写字符串到标准输出：

```asm
    movq    $1, %rax        # SYS_write
    movq    $1, %rdi        # fd = stdout
    leaq    msg(%rip), %rsi
    movq    $len, %rdx
    syscall
```

## 与 call 的差别

| 维度 | `call` / 函数 | `syscall` |
| --- | --- | --- |
| 目标 | 用户态地址（或 PLT） | 内核入口，由 MSR 等配置 |
| 返回 | `ret` 弹栈上返回地址 | 内核恢复用户 `%rip`，经 `%rcx` 保存的返回点 |
| 参数 | SysV：`%rdi`…`%r9`，第四为 `%rcx` | 第四为 `%r10` |
| 破坏 | 按 caller/callee-saved | `%rcx`/`%r11` 被硬件用途占用；其余由内核 ABI 约定 |

libc 的 `write`、`mmap` 等是对上述约定的 C 包装：负责把负数返回值转成 `-1` + `errno`，并处理取消点、重启等策略。反汇编里看到对 `write@plt` 的 `call`，最终仍可能落到带 `syscall` 的 stub。

## vsyscall / vDSO

部分查询类操作（如快速时钟）经 vDSO：内核映射到进程地址空间的用户态可读页，避免完整陷入。对应用而言仍是普通函数调用形态；是否真进内核取决于具体符号与内核版本。阅读性能相关路径时，用 `cat /proc/self/maps` 确认 `[vdso]` 映射是否存在即可，不必在本页展开每个符号。

## 阅读路径

对 libc 包装：

```bash
objdump -d /lib/x86_64-linux-gnu/libc.so.6 | less
# 搜索 write 或 syscall 相关 stub
```

对自写汇编：先确认调用号与参数寄存器，再单步看 `%rax` 返回值。系统调用号表以平台 `unistd_64.h` 或 `ausyscall` 为准；本站示例只用稳定的小数目编号（`read`/`write`/`exit`）。
