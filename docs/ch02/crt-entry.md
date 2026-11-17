# 进程入口与 C 运行时

ELF 可执行文件的入口点是 ELF 头里的 `e_entry`，通常指向 C 运行时（crt）提供的 `_start`，而不是源码里的 `main`。从内核把控制交给用户态，到第一次执行 `main` 的语句，中间还有动态链接器、辅助向量解析与全局构造。本页按这条时间线展开；装载映射细节见 [可执行文件装载](loading.md)。

## 入口符号与 main

`main` 是 C 语言约定的「程序逻辑入口」：它假定 `argc`/`argv`/`envp` 已按 ABI 布置好，`stdin`/`stdout` 已可用，`.init_array` 一类构造已跑完，并且返回值会交给 `exit` 一族做清理。这些前提不是 CPU 复位后的硬件状态，而是用户态启动代码准备出来的。

链接器把入口符号默认设为 `_start`（可通过 `-e` 改写）。手写「只有 `_start` + `syscall`」的最小程序可以绕过 crt，此时没有 libc、没有 `main`，也没有对 `argv` 的 C 层包装——[段与指令](sections-directives.md) 末尾的最小示例即属此类。与 C 混链时，保留 crt 是常态。

## 内核交出控制时的栈

`execve` 成功后，用户态栈顶附近已由内核摆好：

```text
高地址
  … 环境字符串与参数字符串 …
  NULL
  envp[n-1] … envp[0]
  NULL
  argv[argc-1] … argv[0]
  argc
低地址 ← 入口指令执行时 %rsp 附近
```

随后是**辅助向量**（auxiliary vector）：形如 `(type, value)` 的定长对，以 `AT_NULL` 结束。常见项包括 `AT_PHDR` / `AT_PHNUM`（程序头位置）、`AT_ENTRY`（应用入口）、`AT_BASE`（解释器基址）、`AT_RANDOM`（栈金丝雀等用的随机字节）、`AT_PAGESZ`。crt 与动态链接器用这些项定位自身与主程序，而不依赖写死的绝对地址。

## ld.so 与解释器

动态链接的可执行文件在程序头里带 `PT_INTERP`，指向动态链接器路径（常见 `/lib64/ld-linux-x86-64.so.2`）。内核先装载解释器，把控制交给解释器入口；解释器再：

1. 定位主程序的动态段与依赖库；
2. 做必要的重定位，填充 GOT 等；
3. 运行自身与库的初始化例程；
4. 跳转到主程序 ELF 头中的应用入口（常为 crt 的 `_start`），或按约定把入口信息交给后续启动代码。

静态链接可执行文件没有 `PT_INTERP`，内核直接跳到 `e_entry`。无论静动态，**`e_entry` 仍很少等于 `main` 的地址**——它指向启动桩。

## 从 _start 到 main

glibc / musl 的启动路径在细节上不同，用户态可见阶段大致为：

1. `_start`（汇编）从栈上取出 `argc`，算出 `argv`/`envp`，处理辅助向量；
2. 调用 `__libc_start_main`（或等价物），传入 `main` 的地址与构造/析构表；
3. 运行 `.init_array` / 构造函数，初始化 TLS、stdio 等；
4. 调用 `main(argc, argv, envp)`；
5. `main` 返回后走 `exit`，运行 `atexit` 与析构，最终 `exit_group` 系统调用。

入口是 `e_entry`/`_start` 而非 `main`；动态情况下内核先进入 `ld.so`，由它完成库装载与重定位；辅助向量是内核留给用户态的机器可读启动参数，crt 用它找到 PHDR、入口与随机数等，再布置好参数后才调用 `main`。
